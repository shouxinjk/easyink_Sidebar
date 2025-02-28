<template>
  <div>
    <div class="task-wrapper">
      <div class="header">
        <span> 老客标签建群 </span>
        <div> {{ task.createTime }} </div>
      </div>

      <div v-if="task.welcomeMsg" class="guide">
        <div @touchstart.prevent="touchStart" @touchend="touchEnd">
          {{ task.welcomeMsg }}
        </div>
        <div v-show="showCopy" class="copy-wrapper">
          <van-button :class="'copy-btn_' + task.taskId" :data-clipboard-text="task.welcomeMsg"> 复制 </van-button>
        </div>
      </div>

      <div class="content">
        <div>
          <van-image width="50" :src="task.groupCodeInfo.codeUrl" />
        </div>

        <div class="group">
          <span> {{ task.taskName }} </span>
          <!-- <span class="group-name"> 测试群 </span> -->
        </div>
      </div>

      <div>
        <van-cell title="发送给" is-link :value="showMembers" />
        <van-cell title="已完成成员" is-link :value="showDone" @click="goState(1)" />
        <van-cell title="未完成成员" is-link :value="showTodo" @click="goState" />
      </div>

      <div v-if="!state" class="send-button">
        <van-button type="info" size="mini" @click="send">发送</van-button>
      </div>
    </div>

    <div class="bottom-line" />
  </div>
</template>

<script>
import ClipboardJS from 'clipboard';
import { changeStatus } from '@/api/community';
import { TOUCH_DELAY } from '@/utils/constants';
const MARK_LENGTH = 2;

export default {
  name: 'OldCustomerPanel',

  props: {
    task: {
      type: Object,
      required: true
    },

    // true: 已处理  false: 未处理
    state: {
      type: Boolean,
      required: true
    },

    isAdmin: {
      type: Boolean,
      default: true
    }
  },

  data() {
    return {
      showCopy: false, // 展示复制按钮
      touchDelay: TOUCH_DELAY, // 触发显示按钮的长按时常
      copyEvent: null,
      touch: false
    };
  },

  computed: {
    todoMembers() {
      if (!(this.task && this.task.scopeList && this.task.scopeList.length > 0)) return [];

      const members = [];
      for (const member of this.task.scopeList) {
        if (!member.done) members.push(member);
      }

      return members;
    },

    showTodo() {
      if (this.todoMembers.length === 0) return '无';

      const names = this.todoMembers.map(m => m.name);

      if (this.todoMembers.length <= MARK_LENGTH) return names.join('、');
      return names[0] + '、' + names[1] + '等' + this.todoMembers.length + '人';
    },

    doneMembers() {
      if (!this.task || !this.task.scopeList || this.task.scopeList.length === 0) return [];

      const members = [];
      for (const member of this.task.scopeList) {
        if (member.done) members.push(member);
      }

      return members;
    },

    showDone() {
      if (this.doneMembers.length === 0) return '无';

      const names = this.doneMembers.map(m => m.name);

      if (this.doneMembers.length <= MARK_LENGTH) return names.join('、');
      return names[0] + '、' + names[1] + '等' + this.doneMembers.length + '人';
    },

    showMembers() {
      if (!(this.task && this.task.scopeList && this.task.scopeList.length > 0)) return '无';

      if (this.task.scopeList.length === 0) return '无';

      const names = this.task.scopeList.map(m => m.name);

      if (this.task.scopeList.length <= MARK_LENGTH) return names.join('、');

      return names[0] + '、' + names[1] + '等' + this.task.scopeList.length + '人';
    },

    groupCodeUrl() {
      if (window.location.hash[0] === '#') {
        return window.location.origin + window.location.pathname + '#/groupCode?id=' + this.task.groupCodeInfo.id;
      }

      return window.location.origin + window.location.pathname + 'groupCode?id=' + this.task.groupCodeInfo.id;
    }
  },

  mounted() {
    this.clipboard = new ClipboardJS('.copy-btn_' + this.task.taskId);

    this.clipboard.on('success', (e) => {
      this.showCopy = false;
    });

    this.clipboard.on('error', (e) => {
      this.showCopy = false;
    });
  },

  methods: {
    goState(active) {
      if (!this.isAdmin) return;

      this.$router.push({
        name: 'taskState',
        query: {
          taskId: this.task.taskId,
          active: active === 1 ? 1 : 0,
          taskType: 1
        }
      });
    },

    /**
     * 点击发送
     */
    send() {
      const userId = this.$store.state.userId;
      const taskId = this.task.taskId;
      const task = this.task;
      const welcomeMsg = this.task.welcomeMsg;
      const groupCodeUrl = this.groupCodeUrl;
      const _this = this;
      wx.invoke('shareToExternalContact', {
        text: {
          content: welcomeMsg // 回填到分享框的文本内容
        },
        attachments: [
          {
            msgtype: 'link', // 消息类型，必填
            link: {
              title: task.taskName, // H5消息标题
              imgUrl: '', // H5消息封面图片URL
              desc: '客户群活码', // H5消息摘要
              url: groupCodeUrl // H5消息页面url 必填
            }
          }
        ] }, function(res) {
        if (res.err_msg === 'shareToExternalContact:ok') {
          changeStatus(userId, taskId, 0).then(() => {
            _this.$emit('refresh');
          }).catch(() => {
            _this.$emit('refresh');
          });
        }
      }
      );
    },

    touchStart() {
      clearTimeout(this.copyEvent);

      this.copyEvent = setTimeout(() => {
        this.touch = true;
      }, this.touchDelay);
    },

    touchEnd() {
      clearTimeout(this.copyEvent);
      if (this.touch) this.showCopy = true;
      this.touch = false;
    }
  }
};

</script>

<style lang="less" scoped>
  .task-wrapper {
    padding: 10px 8px;

    .header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin: 5px 0 12px;

      span {
        font-size: 18px;
        font-weight: bold;
      }
    }

    .guide {
      padding: 15px;
      background-color: #f2f2f2;
      font-size: 16px;
      line-height: 20px;
      color: #999999;
      margin-bottom: 15px;
      min-height: 30px;
      position: relative;

      .copy-wrapper {
        position: absolute;
        width: 100%;
        height: 100%;
        left: 0;
        top: 0;
        background-color: rgba(0, 0, 0, 0.4);
        display: flex;
        align-items: center;
        justify-content: space-around;
      }
    }

    .content {
      padding: 10px;
      background-color: #f2f2f2;
      display: flex;

      .group {
        padding: 0 10px;
        display: flex;
        flex-direction: column;
        justify-content: space-between;

        .group-name {
          color: #999999;
        }
      }
    }

    .van-cell {
      padding: 10px 6px;
    }

    .send-button {
      padding: 10px 5px 5px;
      display: flex;
      flex-direction: row-reverse;

      .van-button {
        padding: 0 12px;
        margin-right: 12px;
        border-radius: 4px;
        font-size: 12px;
      }
    }
  }

  .bottom-line {
    background-color: #f2f2f2;
    height: 15px;
  }
</style>
