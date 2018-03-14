<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
Vue实现导航权限控制
2018年03月14日

需求：只展示某个登录用户有权限的导航菜单。

基本思路：获取到登录用户可访问的导航数据 → 生成提供给导航组件sideNav.vue的数据 → 处理没有权限的访问路径、令其跳转到403页面。

跟后端约定好的登录返回数据：
```
    // login.json
    {
        "code": 0, // code >= 0 表示成功
        "msg": "success",
        "body": {
            "data": {
                "id": null,
                "name": null,
                "isDisplay": true,
                "operateList": [{
                    "value": null
                }],
                "childsList": [{ // 一级菜单
                    "id": "2",
                    "parentId": null,
                    "name": "内容管理",
                    "operateList": [{ 
                        "value": null
                    }],
                    "childsList": [{ // 二级菜单
                        "id": "21",
                        "parentId": "2",
                        "name": "内容搜索",
                        "operateList": [{
                            "value": "/content/general/search" // 叶子菜单对应的跳转url
                        }]
                    }]
                }]
            }
        }
    }
```


步骤一，获取用户权限数据：
```
    // App.vue
    <script>
    import api from 'api'
    
    export default {
        name: 'App',
        methods: {
        	getRouter () {
                api.getUserInfoAndMenu().then(res => {
                    let menu = res.data || {} // 用户权限数据
                })
        	}
    	},
        created () {
            this.getRouter()
        }
    }
    </script>
```


步骤二，处理获得的用户权限数据，生成菜单数据：
```
    // App.vue
    // ...
    getRouter () {
        api.getUserInfoAndMenu().then(res => {
            let menu = res.data || {} 
            let actualMenu = menu.childsList || []
            let sideMenus = this.productRouterAndMenu(actualMenu)
        })
    },
    productMenu (menu) {
        if (Array.isArray(menu)) {
            let menuItem = menu.map(item => {
                let url = item.operateList[0].value
                return {
                    name: item.name || '',
                    url: url || '',
                    subMenus: this.productRouterAndMenu(item.childsList)
                }
            })
            return menuItem
        }
    }
```

得到了菜单数据：
```
    sideMenus: [{
        'name': '生产者管理',
        'url': '',
        'subMenus': [{
            'name': '权限管理',
            'url': '/content/admin/authority',
            'subMenus': undefined
        }]
    }, {
        'name': '内容管理',
        'url': '',
        'subMenus': [{
            'name': '内容搜索',
            'url': '/content/general/search',
            'subMenus': undefined
        }]
    }]
```


步骤三，把得到的菜单数据存到store实例的state中：
```
    // App.vue
    // ...
    getRouter () {
        api.getUserInfoAndMenu().then(res => {
            let menu = res.data || {} 
            let actualMenu = menu.childsList || []
            let sideMenus = this.productRouterAndMenu(actualMenu)
            // 更新菜单
            this.$store.dispatch('getSideMenus', sideMenus)
        })
    }
    
    // store/actions.js
    export const getSideMenus = ({ commit }, sideMenus) => {
        commit(types.SIDE_MENUS, sideMenus)
    }
    export default {
        // ...
        getSideMenus
    }
    
    // store/mutation-types.js
    export default {
        // ...
        SIDE_MENUS: 'SIDE_MENUS'
    }
    
    // store/mutations.js
    export default {
        // ...
        [types.SIDE_MENUS] (state, sideMenus = []) {
            state.sideMenus = sideMenus
        }
    }
    
    // store/state.js
    export default {
        // ...
        sideMenus: []
    }
```

至此，导航组件 sideNav.vue 就可以拿到菜单数据 sideMenus 了。



到这里，我们的页面已经渲染除了需要呈现的菜单，现在配置下路由信息：
```
    // router/fullRouter.js
    // 完整的路由配置
    import Page from '@/pages/page'
    import Dashboard from '@/pages/dashboard'
    import Error404 from '@/pages/error/404'
    import Error403 from '@/pages/error/403'
    import AuthorityManagement from '@/pages/admin/authorityManagement/authorityManagement'
    import ContentSearch from '@/pages/general/contentSearch/contentSearch'
    
    let fullRouter = [{
        path: '',
        name: 'home',
        component: Page,
        children: [{
            path: '/content/admin',
            name: '生产者管理',
            redirect: '/content/admin/authority',
            component: Dashboard,
            children: [{
                path: '/content/admin/authority',
                name: '权限管理',
                component: AuthorityManagement
            }]
        }, {
            path: '/content/general',
            name: '内容管理',
            redirect: '/content/general/search',
            component: Dashboard,
            children: [{
                path: '/content/general/search',
                name: '内容搜索',
                component: ContentSearch
            }]
        }]
    }, {
        path: '/403',
        name: '403',
        component: Error403
    }, {
        path: '*',
        name: '404',
        component: Error404
    }]
    
    export default fullRouter
    
    
    // router/index.js
    // ... 
    import FullRouter from './fullRouter'
    
    let router = new Router({
        mode: 'history',
        routes: FullRouter
    })
    export default router
```


现在在地址栏手动输入/content/admin/authority，发现可以打开权限管理页面，而返回的用户权限数据没有这个权限，所以我们需要对导航做个[导航守卫](https://router.vuejs.org/zh-cn/advanced/navigation-guards.html)处理。

步骤四，添加导航守卫：
```
    // App.vue
    export default {
        name: 'App',
        data () {
            return {
                routersStr: '/404,/403', // 拼接所有有权限访问的连接
            }
        },
        methods: {
            getRouter () {
                let self = this
                api.getUserInfoAndMenu().then(res => {
                    // ...
                    // 全局导航守卫
                    this.routerGuard()
                })
            },
            productRouterAndMenu (menu) {
                if (Array.isArray(menu)) {
                    let menuItem = menu.map(item => {
                        let url = item.operateList[0].value
                        if (url) {
                            // 拼接所有有权限访问的连接
                            this.routersStr += ',' + url
                        }
                        return {
                            name: item.name || '',
                            url: url || '',
                            subMenus: this.productRouterAndMenu(item.childsList)
                        }
                    })
                    return menuItem
                }
            },
            routerGuard () {
                let self = this
                this.$router.beforeEach((to, from, next) => {
                    // 如果是有权限访问的地址，则跳转；否则跳转到403页面
                    if (self.routersStr.includes(to.path)) {
                        next()
                    } else {
                        next({ path: '/403'})
                    }
                })
            }
        },
        created () {
            this.getRouter()
        }
    }
```

刷新页面、在地址栏手动输入新路由，发现导航守卫没起作用。这是因为手动输入路由回车后，页面重新加载，路由挂载后（此时导航守卫还没挂载）才执行到App.vue里的导航守卫代码，所以导航守卫失效了。

步骤五，那么改用动态添加路由：
```
    // ...
    // 全局导航守卫
    this.routerGuard()
    // 动态添加路由
    this.$router.addRoutes(FullRouter)
```


访问/content/admin/authority（没权限），成功跳转到403页面，看上去我们成功了。但是点击后退键，会再次返回到403页面，再点击一次后退键，才返回到上一个有权限访问的页面。

从现象来看，是没权限页面和403页面都被记录在浏览器历史里了，那么将next方法改为：`next({ path: '/403', replace: true })`， 再次访问/content/admin/authority（没权限）、在403页面点击后退键，成功返回上一个有权限访问的页面。

by Frida




参考：

1. https://juejin.im/entry/59ac970c5188252427260147
2. https://router.vuejs.org/zh-cn/advanced/navigation-guards.html


