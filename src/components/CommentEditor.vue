<template>
  <transition name="dialog-fade" @enter="enter" @enter-to="enterTo" @leave="leave">
    <div autofocus ref="modal" v-show="visible" class="comment-modal" @click.self="close" @keydown.esc.once="close">
      <div class="comment-modal-container" aria-modal="true" :key="key">
        <div class="comment-poster-editor-emoji">
          <VEmojiPicker :pack="pack" @select="selectEmoji" v-show="emojiDialogVisible" labelSearch="搜索表情" />
        </div>
        <div class="comment-poster-container active">
          <ul class="comment-poster-controls">
            <li class="poster-item-close">
              <span class="editor-btn-close" @click="exit">&times;</span>
            </li>
          </ul>
          <div class="comment-poster-main">
            <div class="comment-poster-main-body">
              <img
                v-if="options.comment_gravatar_default"
                class="comment-poster-body-avatar"
                :src="avatar"
                :alt="comment.author"
              />
              <div class="comment-poster-body-content">
                <ul class="comment-poster-body-header">
                  <li class="header-item-nickname">
                    <input
                      type="text"
                      ref="authorInput"
                      v-model="comment.author"
                      @input="handleAuthorInput"
                      placeholder="昵称 *"
                    />
                    <span></span>
                  </li>
                  <li class="header-item-email">
                    <input type="email" v-model="comment.email" placeholder="邮箱 *" />
                    <span></span>
                  </li>
                  <li class="header-item-website">
                    <input type="text" v-model="comment.authorUrl" placeholder="网站" />
                    <span></span>
                  </li>
                </ul>
                <span class="comment-poster-body-reply" v-if="replyingComment"
                  >回复：@{{ replyingComment.author }} <small>#{{ replyingComment.id }}</small></span
                >
                <div class="comment-poster-body-editor">
                  <div class="comment-poster-editor-wrapper">
                    <textarea
                      placeholder="撰写评论...（1000 个字符内）"
                      :style="replyingComment == null ? 'height: 146px;' : 'height: 128px;'"
                      v-model="comment.content"
                      @input="handleContentInput"
                      @focus="() => (this.emojiDialogVisible = false)"
                      ref="contentInput"
                    ></textarea>
                  </div>
                  <ul class="comment-poster-editor-controls">
                    <li class="editor-item-reply">
                      <button
                        class="editor-btn-reply"
                        type="button"
                        @click="handleSubmitClick"
                        :disabled="!commentValid"
                      >
                        评论
                      </button>
                    </li>
                    <li class="editor-item-preview">
                      <button class="editor-btn-preview" type="button" @click="handlePreviewClick">预览</button>
                    </li>
                    <li class="editor-item-emoji">
                      <button class="editor-btn-emoji" type="button" @click="toogleDialogEmoji">表情</button>
                    </li>
                  </ul>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
import md5 from 'md5'
import VEmojiPicker from './EmojiPicker/VEmojiPicker'
import emojiData from './EmojiPicker/data/emojis.js'
import { isEmpty } from '../utils/util'
import commentApi from '../apis/comment'

export default {
  name: 'CommentEditor',
  components: {
    VEmojiPicker,
  },
  model: {
    prop: 'visible',
    event: 'change',
  },
  props: {
    visible: {
      type: Boolean,
      require: true,
    },
    targetId: {
      type: Number,
      required: false,
      default: 0,
    },
    target: {
      type: String,
      required: false,
      default: 'posts',
      validator: function (value) {
        // The value must match one of these strings
        return ['posts', 'sheets', 'journals'].indexOf(value) !== -1
      },
    },
    replyingComment: {
      type: Object,
      required: false,
      default: null,
    },
    options: {
      required: false,
      default: [],
    },
  },
  data() {
    return {
      pack: emojiData,
      emojiDialogVisible: false,
      key: 0,
      modal: null,
      comment: {
        author: null,
        authorUrl: null,
        email: null,
        content: '',
      },
    }
  },
  computed: {
    avatar() {
      const gravatarDefault = this.options.comment_gravatar_default
      const gravatarSource = this.options.gravatar_source || '//cn.gravatar.com/avatar/'

      if (!this.comment.email || !this.validEmail(this.comment.email)) {
        return `${gravatarSource}?d=${gravatarDefault}`
      }

      const gravatarMd5 = md5(this.comment.email)
      return `${gravatarSource}${gravatarMd5}?s=256&d=${gravatarDefault}`
    },
    commentValid() {
      return !isEmpty(this.comment.author) && !isEmpty(this.comment.email) && !isEmpty(this.comment.content)
    },
  },
  created() {
    // Get info from local storage
    this.comment.author = localStorage.getItem('comment-author')
    this.comment.authorUrl = localStorage.getItem('comment-authorUrl')
    this.comment.email = localStorage.getItem('comment-email')

    if (!this.comment.author) {
      this.$nextTick(() => {
        this.$refs.authorInput.focus()
      })
      return
    }
    if (!this.comment.content) {
      this.$nextTick(() => {
        this.$refs.contentInput.focus()
      })
      return
    }
  },
  beforeMount() {
    this.modal = document.createElement('div')
    this.modal.classList.add('v-modal')
  },
  methods: {
    enter() {
      const wrapper = this.$refs.modal
      const zIndex = getComputedStyle(wrapper, false)['zIndex']
      document.body.appendChild(this.modal)
      this.modal.style.zIndex = parseInt(zIndex) - 1
      this.modal.classList.add('v-modal-enter')
    },
    enterTo() {
      this.modal.classList.remove('v-modal-enter')
    },
    leave() {
      this.modal.classList.remove('v-modal-leave')
      document.body.removeChild(this.modal)
    },
    toogleDialogEmoji() {
      this.emojiDialogVisible = !this.emojiDialogVisible
    },
    selectEmoji(emoji) {
      this.comment.content += emoji.emoji
      this.toogleDialogEmoji()
    },
    close() {
      this.$emit('close', false)
    },
    exit() {
      if (this.comment.content && !window.confirm('评论还未发布，是否放弃？')) {
        return
      }
      this.$emit('exit', false)
    },
    handleAuthorInput() {
      this.input()
    },
    handleContentInput() {
      this.input()
    },
    input() {
      this.$emit('input', this.comment)
    },
    handleSubmitClick() {
      // Submit the comment
      this.comment.postId = this.targetId
      if (this.replyingComment) {
        // Set parent id if available
        this.comment.parentId = this.replyingComment.id
      }
      commentApi
        .createComment(this.target, this.comment)
        .then((response) => {
          // Store comment author, email, authorUrl
          if (this.comment.author) {
            localStorage.setItem('comment-author', this.comment.author)
          }
          if (this.comment.email) {
            localStorage.setItem('comment-email', this.comment.email)
          }
          if (this.comment.authorUrl) {
            localStorage.setItem('comment-authorUrl', this.comment.authorUrl)
          }

          // clearn comment
          this.comment.content = null

          // Emit a created event
          this.$emit('created', response.data.data)
          this.$emit('close', false)
        })
        .catch((error) => {
          this.$emit('failed', error.response)
        })
    },
    handlePreviewClick() {
      window.location.href = '#comment-author'
    },
    validEmail(email) {
      var re = /^[A-Za-z1-9]+([-_.][A-Za-z1-9]+)*@([A-Za-z1-9]+[-.])+[A-Za-z]{2,8}$/
      return re.test(email)
    },
  },
}
</script>

<style lang="scss">
.dialog-fade-enter-active {
  -webkit-animation: dialog-fade-in 0.3s;
  animation: dialog-fade-in 0.3s;
}

.dialog-fade-leave-active {
  -webkit-animation: dialog-fade-out 0.3s;
  animation: dialog-fade-out 0.3s;
}

@-webkit-keyframes dialog-fade-in {
  0% {
    -webkit-transform: translate3d(0, -20px, 0);
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }

  100% {
    -webkit-transform: translate3d(0, 0, 0);
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }
}

@keyframes dialog-fade-in {
  0% {
    -webkit-transform: translate3d(0, -20px, 0);
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }

  100% {
    -webkit-transform: translate3d(0, 0, 0);
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }
}

@-webkit-keyframes dialog-fade-out {
  0% {
    -webkit-transform: translate3d(0, 0, 0);
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }

  100% {
    -webkit-transform: translate3d(0, -20px, 0);
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }
}

@keyframes dialog-fade-out {
  0% {
    -webkit-transform: translate3d(0, 0, 0);
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }

  100% {
    -webkit-transform: translate3d(0, -20px, 0);
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }
}
</style>
