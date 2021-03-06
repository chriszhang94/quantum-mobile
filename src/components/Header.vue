<template>
    <div class="header">
        <el-row :gutter="15">
            <el-col :span="2" :lg="{span:1}">
                <div class="collapse-btn" v-on:click="collapseChange">
                    <img :src="imgUrl.logo" style="margin-left: -10px">
                </div>
            </el-col>
            <el-col :span="8" class="quantum-title" :xs="{offset: 2}">
                   <span @click="backHome"> Quantum</span>
            </el-col>
            <el-col :span="2" :lg="{offset: 9}" :xs="{offset:3}" :md="{offset:8}" :sm="{offset:7}">
                <div class="message-container" v-if="showChat">
                    <template v-if="alertNotification.unreadCount > 0">
                        <el-badge :value="alertNotification.unreadCount">
                        <font-awesome-icon icon="bell" @click="toNotification"></font-awesome-icon>
                        </el-badge>
                    </template>
                    <template v-else>
                        <el-badge>
                            <font-awesome-icon icon="bell" @click="toNotification"></font-awesome-icon>
                        </el-badge>
                    </template>
                </div>
            </el-col>
            <el-col :span="2">
                <div class="message-container" v-if="showChat">
                    <template v-if="chatNotification.unreadCount > 0">
                        <el-badge :value="chatNotification.unreadCount">
                            <font-awesome-icon icon="comments" @click="toChat"/>
                        </el-badge>
                    </template>
                    <template v-else>
                        <el-badge>
                            <font-awesome-icon icon="comments" @click="toChat"/>
                        </el-badge>
                    </template>
                </div>
            </el-col>
            <template v-if="showChat">
                <el-col :span="2" align="right">
                    <div class="header-user-con">
                        <el-dropdown class="user-name" trigger="click" @command="handleCommand">
                        <span class="el-dropdown-link">
                            {{username}}
                        </span>
                            <el-dropdown-menu slot="dropdown">
                                <el-dropdown-item command="logout">Logout</el-dropdown-item>
                                <el-dropdown-item command="profile">Profile</el-dropdown-item>
                            </el-dropdown-menu>
                        </el-dropdown>
                    </div>
                </el-col>
            </template>
            <template v-else>
            <el-col :span="2" :xs="{offset:6}" :md="{offset:11}" :sm="{offset:10}" :lg="{offset:12}" align="right">
                <div class="header-user-con">
                    <el-dropdown class="user-name" trigger="click" @command="handleCommand">
                        <span class="el-dropdown-link">
                            {{username}}
                        </span>
                        <el-dropdown-menu slot="dropdown">
                            <el-dropdown-item command="logout">Logout</el-dropdown-item>
                        </el-dropdown-menu>
                    </el-dropdown>
                </div>
            </el-col>
            </template>
<!--        <div class="header-right">-->
<!--            <div class="header-user-con">-->
<!--                <el-dropdown class="user-name" trigger="click" @command="handleCommand">-->
<!--                    <span class="el-dropdown-link">-->
<!--                        {{username}}-->
<!--                        <i class="el-icon-caret-bottom"></i>-->
<!--                    </span>-->
<!--                    <el-dropdown-menu slot="dropdown">-->
<!--                        <el-dropdown-item command="logout">Logout</el-dropdown-item>-->
<!--                    </el-dropdown-menu>-->
<!--                </el-dropdown>-->
<!--            </div>-->
<!--        </div>-->
        </el-row>


    </div>
</template>

<script>
    import {SessionStorage} from "@/utils/SessionStorage";
    import {SESSION_KEY_LOGIN_USER, AUTH_TOKEN} from "@/utils/Constants";
    import {Toast} from "@/utils/Toast";
    import {Confirm} from "@/utils/confirm";

    export default {
        name: "Header",
        props: {
          showChat:{
              type: Boolean
          }
        },
        mounted() {
            if (document.body.clientWidth < 1500) {
                this.collapseChange();
            }
        },
        computed: {
            username(){
                let name = SessionStorage.getJson(SESSION_KEY_LOGIN_USER).username;
                return name == null ? '' : name;
            },
            chatNotification(){
                return this.$store.state.chatNotification;
            },
            alertNotification(){
                return this.$store.state.alertNotification;
            }
        },
        data: function(){
            return {
                collapse: false,
                fullscreen: false,
                message: 2,
                imgUrl: {
                    src: require('../assets/img/quantumLogo_small.png'),
                    logo: require('../assets/img/quantumLogo_whiteMark.png')
                }

            }
        },
        methods: {
            backHome(){
              this.$router.push('/dashboard');
            },
            handleCommand(command){
                if(command === 'logout'){
                    this.logout();
                }else if(command == 'profile'){
                    this.$router.push('/profile')
                }
            },
            collapseChange(){
                this.collapse = !this.collapse;
                this.$bus.emit("collapse", this.collapse);
            },
            logout(){
                Confirm('Are you sure to logout?', () => {
                    SessionStorage.pop(AUTH_TOKEN);
                    SessionStorage.pop(SESSION_KEY_LOGIN_USER);
                    this.$message.success('Logout Successfully!');
                    setTimeout(() => {
                        this.$router.push('/');
                    }, 1000)
                });

            },
            toChat(){
                this.$router.push('/chat')
            },
            toNotification(){
                this.$router.push('/notification')
            }
        }

    }
</script>

<style scoped>
    .message-container{
        margin-top: 10px;
    }
    .header {
        position: relative;
        box-sizing: border-box;
        width: 100%;
        height: 50px;
        font-size: 22px;
        color: rgba(255, 255, 255, 0.99);
        background: black;
    }

    .quantum-title{
        margin-top: 10px;
        float: left;
        text-align: left;
    }

    .quantum-title span{
        font-weight: bolder;
    }
    .collapse-btn {
        float: left;
        padding: 0 21px;
        cursor: pointer;
        line-height: 70px;
    }

    .header .logo {
        line-height: 90px;
    }

    .header-right {
        float: right;
        padding-right: 50px;
    }

    .header-user-con {
        display: flex;
        height: 50px;
        align-items: center;
    }

    .btn-bell .el-icon-bell {
        color: #fff;
    }

    .user-name {
        margin-left: 10px;
    }

    .user-avator {
        margin-left: 20px;
    }

    .user-avator img {
        display: block;
        width: 40px;
        height: 40px;
        border-radius: 50%;
    }

    .el-dropdown-link {
        color: #fff;
        cursor: pointer;
        font-weight: bold;
    }

    .el-dropdown-menu__item {
        text-align: center;
    }

    span{
        font-weight: 200;
    }
</style>
