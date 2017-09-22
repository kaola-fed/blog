---
title: gulp源码解析之任务管理
date: 2017-08-31
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

提到前端工程的自动化构建，gulp是其中很重要的一个工具，gulp是一种基于stream的前端构建工具，相比于grunt使用临时文件的策略，会有较大的速度优势。本文会对gulp的主要部分进行详细剖析，期待本文能够帮助读者更好地在工程中实践gulp。

gulp等前端构建脚本并不是一种全新的思想，早在几十年前，gnu make已经在各种流行语言中风靡，并解决了相关的各种问题，可以简单的认为gulp是JavaScript语言的“make”。

### Gulp 核心模块

gulp其中两大核心模块，任务管理和文件

本文通过对[Orchestrator](https://github.com/robrich/orchestrator)任务管理模块的源码进行分析，理清gulp是如何进行任务管理的。

通过查看[gulp源码（3.x及之前的版本）](https://github.com/gulpjs/gulp/blob/master/index.js)，可以看到，gulp所有任务管理相关的功能都是直接从Orchestrator模块继承的，而且可以发现gulp官网对相关任务管理接口的描述，和Orchestrator模块的描述几乎完全一样。

```javascript
// gulp/index.js
...
var Orchestrator = require('orchestrator');
...

function Gulp() {  // 构造函数直接调用Orchestrator
  Orchestrator.call(this);
}

util.inherits(Gulp, Orchestrator);  // 从Orchestrator继承而来

Gulp.prototype.task = Gulp.prototype.add;  // task方法直接使用Orchestrator中的add
...
```

所以下面就一起来分析一下Orchestrator模块。

### Orchestrator 模块

[Orchestrator github主页](https://github.com/robrich/orchestrator)中对自身的描述

> A module for sequencing and executing tasks and dependencies in maximum concurrency.

简单翻译就是

> 一个能够以最大并发性对任务及其依赖任务进行排序执行的模块。

Orchestrator模块主要就是添加管理任务

```javascript
// Orchestrator/index.js

/*jshint node:true */

"use strict";

var util = require('util');
var events = require('events');
var EventEmitter = events.EventEmitter;
var runTask = require('./lib/runTask');

/*
 构造函数里面
 首先调用node中事件模块的构造函数EventEmitter，
 
 EventEmitter.call(this);
 
 这是一种典型的用法，直接挂载Event模块实例属性及方法。
 
 接下来初始化了Orchestrator最核心的几个属性：
 
 doneCallback 是所有任务执行完成后的回调
 seq 是排序后的任务队列，对任务及其依赖的排序是通过seqencify模块进行的，后面会介绍。
 tasks 保存了所有定义的任务，通过add原型方法添加任务。
 isRunning 标识当前是否有任务正在执行
*/
var Orchestrator = function () {
	EventEmitter.call(this);
	this.doneCallback = undefined; // call this when all tasks in the queue are done
	this.seq = []; // the order to run the tasks
	this.tasks = {}; // task objects: name, dep (list of names of dependencies), fn (the task to run)
	this.isRunning = false; // is the orchestrator running tasks? .start() to start, .stop() to stop
};

/*
 * 继承Event模块
 */
util.inherits(Orchestrator, EventEmitter);

/*
 * 重置Orchestrator模块，即重置相应的状态和属性
 */
Orchestrator.prototype.reset = function () {
	if (this.isRunning) {
		this.stop(null);
	}
	this.tasks = {};
	this.seq = [];
	this.isRunning = false;
	this.doneCallback = undefined;
	return this;
};

/*
  add前面主要是对参数进行了一些校验检测，最后将任务task的名字，依赖，回调添加到核心属性tasks中
*/
Orchestrator.prototype.add = function (name, dep, fn) {
	if (!fn && typeof dep === 'function') {
		fn = dep;
		dep = undefined;
	}
	dep = dep || [];
	fn = fn || function () {}; // no-op
	if (!name) {
		throw new Error('Task requires a name');
	}
	// validate name is a string, dep is an array of strings, and fn is a function
	if (typeof name !== 'string') {
		throw new Error('Task requires a name that is a string');
	}
	if (typeof fn !== 'function') {
		throw new Error('Task '+name+' requires a function that is a function');
	}
	if (!Array.isArray(dep)) {
		throw new Error('Task '+name+' can\'t support dependencies that is not an array of strings');
	}
	dep.forEach(function (item) {
		if (typeof item !== 'string') {
			throw new Error('Task '+name+' dependency '+item+' is not a string');
		}
	});
	this.tasks[name] = {
		fn: fn,
		dep: dep,
		name: name
	};
	return this;
};

/*
  如果只给了task的名字，则直接获取之前定义task，否则通过add添加新的task
*/
Orchestrator.prototype.task = function (name, dep, fn) {
	if (dep || fn) {
		// alias for add, return nothing rather than this
		this.add(name, dep, fn);
	} else {
		return this.tasks[name];
	}
};

/*
  判断是否已经定义了某个task
*/
Orchestrator.prototype.hasTask = function (name) {
	return !!this.tasks[name];
};

// tasks and optionally a callback
Orchestrator.prototype.start = function() {
	var args, arg, names = [], lastTask, i, seq = [];
	args = Array.prototype.slice.call(arguments, 0);
	if (args.length) {
		lastTask = args[args.length-1];
		if (typeof lastTask === 'function') {
			this.doneCallback = lastTask;
			args.pop();
		}
		for (i = 0; i < args.length; i++) {
			arg = args[i];
			if (typeof arg === 'string') {
				names.push(arg);
			} else if (Array.isArray(arg)) {
				names = names.concat(arg); // FRAGILE: ASSUME: it's an array of strings
			} else {
				throw new Error('pass strings or arrays of strings');
			}
		}
	}
	if (this.isRunning) {
		// reset specified tasks (and dependencies) as not run
		this._resetSpecificTasks(names);
	} else {
		// reset all tasks as not run
		this._resetAllTasks();
	}
	if (this.isRunning) {
		// if you call start() again while a previous run is still in play
		// prepend the new tasks to the existing task queue
		names = names.concat(this.seq);
	}
	if (names.length < 1) {
		// run all tasks
		for (i in this.tasks) {
			if (this.tasks.hasOwnProperty(i)) {
				names.push(this.tasks[i].name);
			}
		}
	}
	seq = [];
	try {
		this.sequence(this.tasks, names, seq, []);
	} catch (err) {
		// Is this a known error?
		if (err) {
			if (err.missingTask) {
				this.emit('task_not_found', {message: err.message, task:err.missingTask, err: err});
			}
			if (err.recursiveTasks) {
				this.emit('task_recursion', {message: err.message, recursiveTasks:err.recursiveTasks, err: err});
			}
		}
		this.stop(err);
		return this;
	}
	this.seq = seq;
	this.emit('start', {message:'seq: '+this.seq.join(',')});
	if (!this.isRunning) {
		this.isRunning = true;
	}
	this._runStep();
	return this;
};


Orchestrator.prototype.stop = function (err, successfulFinish) {
	this.isRunning = false;
	if (err) {
		this.emit('err', {message:'orchestration failed', err:err});
	} else if (successfulFinish) {
		this.emit('stop', {message:'orchestration succeeded'});
	} else {
		// ASSUME
		err = 'orchestration aborted';
		this.emit('err', {message:'orchestration aborted', err: err});
	}
	if (this.doneCallback) {
		// Avoid calling it multiple times
		this.doneCallback(err);
	} else if (err && !this.listeners('err').length) {
		// No one is listening for the error so speak louder
		throw err;
	}
};

/*
 * 引入任务及其依赖的排序模块
 */
Orchestrator.prototype.sequence = require('sequencify');

/*
  简单的循环判断是否所有的任务都已经执行完毕
*/
Orchestrator.prototype.allDone = function () {
	var i, task, allDone = true; // nothing disputed it yet
	for (i = 0; i < this.seq.length; i++) {
		task = this.tasks[this.seq[i]];
		if (!task.done) {
			allDone = false;
			break;
		}
	}
	return allDone;
};

/*
 * 重置task，重置相应的状态和属性
 */
Orchestrator.prototype._resetTask = function(task) {
	if (task) {
		if (task.done) {
			task.done = false;
		}
		delete task.start;
		delete task.stop;
		delete task.duration;
		delete task.hrDuration;
		delete task.args;
	}
};

/*
 * 循环遍历重置所有的task
 */
Orchestrator.prototype._resetAllTasks = function() {
	var task;
	for (task in this.tasks) {
		if (this.tasks.hasOwnProperty(task)) {
			this._resetTask(this.tasks[task]);
		}
	}
};

/*
 * 递归重置task及其依赖task
 */
Orchestrator.prototype._resetSpecificTasks = function (names) {
	var i, name, t;

	if (names && names.length) {
		for (i = 0; i < names.length; i++) {
			name = names[i];
			t = this.tasks[name];
			if (t) {
				this._resetTask(t);
				if (t.dep && t.dep.length) {
					this._resetSpecificTasks(t.dep); // recurse
				}
			//} else {
				// FRAGILE: ignore that the task doesn't exist
			}
		}
	}
};

Orchestrator.prototype._runStep = function () {
	var i, task;
	if (!this.isRunning) {
		return; // user aborted, ASSUME: stop called previously
	}
	for (i = 0; i < this.seq.length; i++) {
		task = this.tasks[this.seq[i]];
		if (!task.done && !task.running && this._readyToRunTask(task)) {
			this._runTask(task);
		}
		if (!this.isRunning) {
			return; // task failed or user aborted, ASSUME: stop called previously
		}
	}
	if (this.allDone()) {
		this.stop(null, true);
	}
};

Orchestrator.prototype._readyToRunTask = function (task) {
	var ready = true, // no one disproved it yet
		i, name, t;
	if (task.dep.length) {
		for (i = 0; i < task.dep.length; i++) {
			name = task.dep[i];
			t = this.tasks[name];
			if (!t) {
				// FRAGILE: this should never happen
				this.stop("can't run "+task.name+" because it depends on "+name+" which doesn't exist");
				ready = false;
				break;
			}
			if (!t.done) {
				ready = false;
				break;
			}
		}
	}
	return ready;
};

Orchestrator.prototype._stopTask = function (task, meta) {
	task.duration = meta.duration;
	task.hrDuration = meta.hrDuration;
	task.running = false;
	task.done = true;
};

Orchestrator.prototype._emitTaskDone = function (task, message, err) {
	if (!task.args) {
		task.args = {task:task.name};
	}
	task.args.duration = task.duration;
	task.args.hrDuration = task.hrDuration;
	task.args.message = task.name+' '+message;
	var evt = 'stop';
	if (err) {
		task.args.err = err;
		evt = 'err';
	}
	// 'task_stop' or 'task_err'
	this.emit('task_'+evt, task.args);
};

Orchestrator.prototype._runTask = function (task) {
	var that = this;

	task.args = {task:task.name, message:task.name+' started'};
	this.emit('task_start', task.args);
	task.running = true;

	runTask(task.fn.bind(this), function (err, meta) {
		that._stopTask.call(that, task, meta);
		that._emitTaskDone.call(that, task, meta.runMethod, err);
		if (err) {
			return that.stop.call(that, err);
		}
		that._runStep.call(that);
	});
};

// FRAGILE: ASSUME: this list is an exhaustive list of events emitted
var events = ['start','stop','err','task_start','task_stop','task_err','task_not_found','task_recursion'];

var listenToEvent = function (target, event, callback) {
	target.on(event, function (e) {
		e.src = event;
		callback(e);
	});
};

Orchestrator.prototype.onAll = function (callback) {
	var i;
	if (typeof callback !== 'function') {
		throw new Error('No callback specified');
	}

	for (i = 0; i < events.length; i++) {
		listenToEvent(this, events[i], callback);
	}
};

module.exports = Orchestrator;
```

#### sequencify

sequencify模块主要完成的工作就是对一个给定的任务数组，分析依赖关系，给出一个按照任务依赖顺序排列的任务数组，然后依次执行


```javascript
// sequencify/index.js


```

对于gulp 4.x之后的版本，gulp更换了任务管理的模块为undertaker，后续文章中会对其进一步分析，并与orchestrator进行一定的对比。

接下来的系列文章会对gulp中的stream流管理进行分析。

待更新
