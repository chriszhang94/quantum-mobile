<template>
    <div>
        <div class="chatroom-container">
            <div class="discussion-filter">
                <el-row class="margin-top10">
                    <el-col>
                <AutoCompleteInput
                            class="team-task-discussion private-chat-user-input"
                            placeholder="Search for users"
                            :existing-tags="existing_tags"
                            :typeahead-url="typeahead_url"
                            :user-id="userId"
                            typeahead-style="dropdown"
                            :typeahead-hide-discard="true"
                            :typeahead-max-results=5
                            :limit="5"
                            :hide-input-on-limit="true"
                            @tag-added="onTagAdded"
                            @tag-removed="onTagRemoved"
                            v-bind:wrapper-class="'tags-input-wrapper-custom'"
                            :typeahead="true"
                ref="autocomplete">
                </AutoCompleteInput>
                <img class="add-discussion-recipient-input"
                     style="margin-right: 5px"
                     :src="imgUrl.src"
                     v-show="userSearch.length > 0"
                     @click="createNewChatGroup"
                   >
                    </el-col>
                    </el-row>
            </div>
            <div class="chat-room-panel overflow-y-auto" :style="{height: this.windowHeight - 110 + 'px'}">
                <template v-if="room_loaded">
                    <el-row
                            v-for="(chatRoom,index) in chatRooms" :key="index">
                        <Roomblock
                                :index="index"
                                v-on:openroom="openRoom"
                        ></Roomblock>
                    </el-row>
                </template>
            </div>
        </div>
    </div>
</template>

<script>
    import Roomblock from "../components/Roomblock";
    import AutoCompleteInput from "../components/AutoCompleteInput";
    import {SessionStorage} from "@/utils/SessionStorage";
    import {SESSION_KEY_LOGIN_USER, AUTH_TOKEN, CHATROOM, NEW_CHAT_MESSAGE} from "@/utils/Constants";
    import {getAllRooms, createRoom} from "@/quantumApi/chat/chat";
    import {Toast} from "@/utils/Toast";

    export default {
        name: "Chat",
        components: {Roomblock, AutoCompleteInput},
        data(){
            return{
                userSearch: [],
                auth: null,
                userId: null,
                companyId: null,
                typeahead_url: process.env.VUE_APP_SERVER + 'chat/get_users_in_company/?name=:search&user_id=:userID',
                testRooms:[{
                    room_id: 10,
                    name: 'TestRoom',
                    message: {
                        from: 'Justin Trisdale',
                        timestamp: '1598584947000',
                        body: 'hello'
                    },
                    messages:{},
                    unreadCount: 0
                }],
                existing_tags: [{full_name: ''}],
                imgUrl: {
                    src: require('../assets/img/create.png'),
                },
                pendingMessages: {},
                unSentMessages: [],
                searchText: '',
                room_loaded: false,
                message: 'Init',
                chatRoomWindow: {
                    mainWindowShown: false,
                    minimized: false,
                    activeWindow: null
                },
                serverUrl: "",
                discussionTabs: [{
                    title: 'Chat',
                    type: 'users',
                    unreadCount: 0,
                },
                ],
                currentTab: 'users',
                DISCUSSION_CONTAINER: '.discussion-scroll-div',
                ROOMS_PER_PAGE: 20,
                openedRooms: [],
                roomParameters: {
                    userSearchString: [],
                    roomSearchString: [],
                    at: false,
                    imp: false,
                    hasMoreRooms: true,
                    loading: false,
                    membersForNewGroup: []
                },
                workroomChatRoom: {
                    room_id: null,
                    unreadCount: 0,
                    messages: [],
                    disabled: false
                },
                MESSAGE_PAGINATE_ROWS: 20,
                selected_room: null,
                windowHeight: window.innerHeight
            }
        },
        computed: {
            chatRooms(){
                return this.$store.state.chatRooms
            },
            displayedRooms(){
                return this.$store.state.displayedRooms
            }

        },
        watch: {
            chatRooms:{
                handler(oldValue, newValue){
                }
            }
        },
        created(){
            this.initParams();
            this.getRooms();
        },
        mounted(){
            let vm = this;
            this.$bus.on(NEW_CHAT_MESSAGE, (msg) => {
                this.newMessageReceived(msg);
            });
        },
        beforeDestroy(){
            let vm = this;
            this.$bus.off(NEW_CHAT_MESSAGE)
        },
        methods: {
            newMessageReceived(messages){
                let message, room, index;
                for(index = 0;index < messages.length;index++){
                    message = messages[index];
                    let room = this.findInArray(this.chatRooms, message.room_id, 'room_id');
                    if(room){
                        this.$set(room, 'message', message);
                        if(!message.read){
                            this.unreadCountChange(room, 1);
                        }
                        this.$store.commit('unshiftchatRoom', room);
                    }
                }
            },
            unreadCountChange(chatRoom, newCount){
                let newcount = chatRoom.unreadCount + newCount;
                this.$set(chatRoom, 'unreadCount', newcount);
                this.$store.commit('updatedisplayedRooms', chatRoom);
                this.$store.commit('updatechatRooms', chatRoom);
            },
            findInArray: function(array, target, field, returnIndex){
                field = field || 'id';
                for(let index=0; index < array.length; index++){
                    if(array[index][field] == target)
                        return returnIndex ? index : array[index];
                }
                return returnIndex ? -1 : null;
            },
            onTagAdded(slug){
                this.userSearch.push(slug);
            },
            async getRooms(loadMore, callBack){
                let vm = this;
                if(vm.roomParameters.loading){
                    return;
                }
                if(!loadMore){
                    vm.roomParameters.hasMoreRooms = true;
                    vm.chatRoomMap = {};
                }else if(!vm.roomParameters.hasMoreRooms){
                    return;
                }
                vm.roomParameters.loading = true;
                let filters = [];
                let chatRooms = vm.$store.state.chatRooms;
                let postData = {
                    user_id: this.userId,
                    company_id: this.companyId,
                    discussion_filters: JSON.stringify(filters),
                    room_type: vm.currentTab,
                    rows: vm.ROOMS_PER_PAGE,
                    term: vm.roomParameters.roomSearchString,
                    exclude_room_ids: JSON.stringify(loadMore ? this.getAttrs(chatRooms, 'room_id') : [])
                };
                if(!loadMore){
                    vm.$store.commit('setEmptyRoom');
                }
                let resp = await getAllRooms(postData, this.auth);
                let data = resp.data;
                vm.roomParameters.loading = false;
                vm.room_loaded = true;
                let newRooms = data.rooms;
                for(let i=0;i<newRooms.length;i++){
                    let room = newRooms[i];
                    vm.initChatRoom(room);
                }
                vm.roomParameters.hasMoreRooms = data.has_more_rooms;
                vm.$store.commit('concatRoom', newRooms);
                vm.updateRoomMap(data.rooms);
                if(callBack){
                    callBack();
                }
            },
            updateRoomMap: function(rooms){
                let room;
                let vm = this;
                for(let i=0;i<rooms.length;i++){
                    room = rooms[i];
                    room.unreadCount = room.unread_count;
                    vm.chatRoomMap[room.room_id] = room;
                    this.updateWorkroomChatRoomCount(room.room_id, room.unreadCount);
                }
            },
            isRoomActive: function(room_id){
                let vm = this;
                return vm.chatRoomWindow.activeWindow &&
                    room_id == vm.chatRoomWindow.activeWindow.room_id &&
                    ! vm.chatRoomWindow.activeWindow.minimized;
            },
            updateWorkroomChatRoomCount: function(room_id, unreadCount, countDiff){
                let vm = this;
                if(!vm.isRoomActive(room_id)){
                    return;
                }
                if(vm.workroomChatRoom.room_id == room_id){
                    if(unreadCount !== null){
                        vm.workroomChatRoom.unreadCount = unreadCount;
                    }else{
                        vm.workroomChatRoom.unreadCount += countDiff;
                    }
                }
            },
            initChatRoom(chatRoom){
                this.$set(chatRoom, 'loadMessages', false);
                this.$set(chatRoom, 'newMembers', []);
                this.$set(chatRoom, 'messages', []);
                this.$set(chatRoom, 'oldMessageCount', undefined);
                this.$set(chatRoom, 'newMessageReceived', false);
                this.$set(chatRoom, 'minimized', false);
                this.$set(chatRoom, 'searchText', '')
                this.$set(chatRoom, 'searchResult', {
                    ROWS: 20,
                    ROWS_PER_TIME: 5,
                    remainCounts: 0,
                    totalCounts: 0,
                    shownMessages: null,
                    unShownMessages: null
                })
            },
            initParams(){
                let user = SessionStorage.getJson(SESSION_KEY_LOGIN_USER);
                this.userId = user.user_id;
                this.companyId = user.company_id;
                this.auth = SessionStorage.get(AUTH_TOKEN);
            },
            getAttrs: function(array, field){
                let newArray = [];
                for(let i=0; i<array.length;i++){
                    newArray.push(array[i][field]);
                }
                return newArray;
            },
            openRoom(chatRoom){
                SessionStorage.setJSON(CHATROOM, chatRoom);
                this.$router.push('/chatroom');
            },
            onTagRemoved(slug){
                let index = null;
                for(let i=0;i<this.userSearch.length;i++){
                    if(this.userSearch[i] == slug){
                        index = i;
                        break;
                    }
                }
                this.userSearch.splice(index, 1);
            },
            createNewChatGroup(){
                let ids = this.getUserIds(this.userSearch);
                this.userSearch = [];
                this.$refs['autocomplete'].clearTags()
                this.createNewChatWithIds(ids);
            },
            async createNewChatWithIds(userIds){
                if(userIds.length == 0){
                    return;
                }
                let params = {
                    'user_ids': JSON.stringify(userIds)
                }
                let room;
                let resp = await createRoom(params, this.auth);
                if(resp.status === 200){
                    let data = resp.data;
                    if(data.success){
                        if(data.new){
                            room = data.room;
                            this.$store.commit('unshiftchatRoom', room);
                            Toast.success('New Chat Room Created!');
                        }else{
                            room = data.room;
                            room = this.findInArray(this.$store.state.chatRooms, room.room_id, 'room_id');
                            this.$store.commit('unshiftchatRoom', room);
                            Toast.success('Room ' + room.name + ' already exists!');
                        }
                    }else{
                        Toast.error(data.message);
                    }
                }else{
                    Toast.error(resp);
                }
            },
            getUserIds: function(groups){
                let ids = []
                for(let i=0;i<groups.length;i++){
                    ids.push(groups[i].id);
                }
                return ids;
            },
        }
    }
</script>

<style>
    .overflow-y-auto {
        overflow-y: auto;
    }

    .height-250 {
        height: 250px;
    }
    .chatroom-container{
        display: block;
        flex: 1;
        flex-basis: auto;
        overflow: auto;
        box-sizing: border-box;
        background-color: #ffffff;
        height: auto;
        box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2), 0 6px 20px 0 rgba(0, 0, 0, 0.19);
    }
    .discussion-filter{
        padding: 2px 0;
        min-height: 60px;
        border-bottom: 1px solid #dddddd;
    }
    .private-chat-user-input {
        font-size: 14px;
        display: inline-block;
        border: none;
        border-radius: 15px;
        background-color: #f1f1f1;
        box-shadow: none;
        cursor: text;
        min-height: 30px;
        width: 80%;
    }
    .tags-input{
        padding: 10px 10px;
    }
    .add-discussion-recipient-input {
        height: 18px;
        cursor: pointer;
        margin-top: 10px;
        float: right;
    }
</style>
