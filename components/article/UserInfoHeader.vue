<template>
  <div class="Post-Author">
    <div class="AuthorInfo">
      <c-user-popover :user-id="Number(article.uid)">
        <router-link
          :to="{name: 'user-id', params: {id : article.uid}}"
          target="_blank"
        >
          <c-avatar
            :src="avatarSrc"
            class="Avatar"
            :recommend-author="user.is_recommend === 1"
            :level="1"
            :token-user="!!tokenInfo.id"
            :level-token="1"
          />
        </router-link>
      </c-user-popover>
      <div class="AuthorInfo-content">
        <router-link class="UserLink AuthorInfo-name" :to="`/user/${article.uid}`" target="_blank">
          {{ avatarName || '&nbsp;' }}
        </router-link>
        <div class="fl ac author-info">
          <span class="Post-Time">{{ time }}</span>
          <span class="View-Num"><svg-icon class="icon" icon-class="read" />{{ article.read || 0 }}</span>
          <ipfsAll :article-ipfs-array="articleIpfsArray" :user="user" :github-id="githubId" />
          &nbsp;
          <span class="article-head__ipfs">{{ isPublishedOnGithub ? 'GitHub' : 'IPFS' }}</span>
        </div>
      </div>
    </div>
    <template v-if="!isMe(article.uid)">
      <el-button
        :class="!info.is_follow && 'black'"
        size="small"
        class="follow"
        @click.stop="followOrUnFollow"
      >
        <i
          v-if="!info.is_follow"
          class="el-icon-plus"
        />
        {{ followBtnText }}
      </el-button>
    </template>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import ipfsAll from '@/common/components/ipfs_all/index.vue'

export default {
  components: {
    ipfsAll
  },
  props: {
    article: {
      type: Object,
      required: true
    },
    articleIpfsArray: {
      type: Array,
      required: true
    },
    isHide: {
      type: Boolean,
      required: false
    }
  },
  data() {
    return {
      avatarSrc: '',
      info: {
        is_follow: 0 // 默认值
      },
      user: Object.create(null), // 用户信息
      tokenInfo: Object.create(null), // token info,
      githubId: ''
    }
  },
  computed: {
    ...mapGetters(['isLogined', 'isMe']),
    hash() {
      return this.articleIpfsArray.length !== 0
        ? this.articleIpfsArray[0].htmlHash
        : ''
    },
    isPublishedOnGithub() {
      return this.hash.startsWith('Gh')
    },
    avatarName() {
      // 因为内容比较宽 没有控制12字符
      const name = this.article.nickname || this.article.username
      return name.length > 24 ? name.slice(0, 24) + '...' : name
    },
    followBtnText() {
      return this.info.is_follow ? this.$t('following') : this.$t('follow')
    },
    time() {
      const { create_time: createTime } = this.article
      return createTime ? this.moment(createTime).format('YYYY-MM-DD HH:mm') : ''
    }
  },
  watch: {
    article() {
      // 完成获取用户信息
      this.getUserInfo(this.article.uid)
      this.getTokenUser(this.article.uid)
    }
  },
  async mounted() {
    // 完成获取用户信息
    this.getUserInfo(this.article.uid)
    await this.getTokenUser(this.article.uid)
    await this.getGithubId()
  },
  methods: {
    // 主要获取关注状态
    getUserInfo(id) {
      this.$API.getUser(id).then(res => {
        if (res.code === 0) {

          this.user = res.data

          this.info.is_follow = res.data.is_follow
          this.avatarSrc = res.data.avatar ? this.$ossProcess(res.data.avatar) : ''
        } else {
          this.$message({ showClose: true, message: res.message, type: 'warning'})
        }
      }).catch(err => {
        console.log(`获取关注状态失败${err}`)
      })
    },
    async getTokenUser(id) {
      // 获取是否有 token
      const tokenRes = await this.$utils.factoryRequest(this.$API.tokenUserId(id))
      // not token, data is null
      this.tokenInfo = tokenRes ? (tokenRes.data || {}) : {}
    },
    // 获取 GitHub ID
    async getGithubId() {
      try {
        const res = await this.$API.getIpfsData(this.article.hash)
        if (res.code !== 0) {
          this.$message.error(res.message)
          return
        }
        this.githubId = res.data.github_id
      } catch (e) {
        this.$message.error(e.message)
      }
    },
    followOrUnFollow() {
      if (this.info.is_follow) {
        this.$confirm(this.$t('p.confirmUnFollowMessage'), this.$t('promptTitle'), {
          confirmButtonText: this.$t('confirm'),
          cancelButtonText: this.$t('cancel'),
          type: 'warning'
        }).then(() => {
          this.followOrUnfollowUser(this.article.uid, 0)
        })
      } else {
        this.followOrUnfollowUser(this.article.uid, 1)
      }
    },
    async followOrUnfollowUser(id, type) {
      if (!this.isLogined) return this.$store.commit('setLoginModal', true)
      const message = type === 1 ? this.$t('follow') : this.$t('unFollow')
      try {
        let res = null
        if (type === 1) res = await this.$API.follow(id)
        else res = await this.$API.unfollow(id)
        if (res.code === 0) {
          this.$message({ showClose: true, message: `${message}${this.$t('success.success')}`, type: 'success'})
          this.info.is_follow = type === 1
          // 在获取一次防止出错
          this.getUserInfo(this.article.uid)
        } else {
          this.$message({ showClose: true, message: `${message}${this.$t('error.fail')}`, type: 'error' })
        }
      } catch (error) {
        this.$message({ showClose: true, message: `${message}${this.$t('error.fail')}`, type: 'error' })
      }
    }
  }
}
</script>

<style scoped lang="less">
.follow:active {
  transform: scale(0.9);
  box-shadow: 0 2px 25px rgba(163, 163, 163, 0.747);
}
.Avatar {
  width: 50px !important;
  height: 50px !important;
  /deep/ .recommend-icon {
    right: -10px !important;
    bottom: -2px !important;
    width: 26px !important;
  }
}
.AuthorInfo-content {
  margin: 0 10px;
}
.Post-Author {
  display: flex;
  align-items: center;
  .AuthorInfo {
    flex: 1 1;
    white-space: nowrap;
    display: flex;
    align-items: center;
  }
  .AuthorInfo-content {
    overflow: hidden;
  }
  .AuthorInfo-name {
    font-size: 18px;
    font-weight: 500;
    color: rgba(0, 0, 0, 1);
    margin: 0 0 4px 0;
    display: inline-block;
  }
}
.Post-Time {
  font-weight: 400;
  color: rgba(178, 178, 178, 1);
}
.View-Num {
  font-weight: 400;
  color: rgba(178, 178, 178, 1);
  margin: 0 10px;
  display: flex;
  align-items: center;
  font-size: 14px;
  .icon {
    font-size: 18px;
    margin-right: 6px;
  }
}
.Post-Time,
.View-Num {
  font-size: 16px;
  color: @gray;
}
.follow {
  margin: 0 0 0 20px;
  &.black {
    background: #333;
    color: #fff;
    border: 1px solid #333;
  }
}

.article-head__ipfs {
  font-size: 16px;
  font-weight: bold;
  color: rgba(84, 45, 224, 1);
  line-height: 14px;
}

.author-info {
  flex-wrap: wrap;
}
@media screen and (max-width: 600px) {
  .Post-Author .Avatar {
    /deep/ .c-avatar {
      width: 30px !important;
      height: 30px !important;
    }
  }
  .Post-Author .AuthorInfo-name {
    font-size: 16px;
    margin: 0;
  }
  .Post-Time,
  .View-Num {
    font-size: 12px;
  }
  .article-head__ipfs {
    font-size: 12px;
  }
  /deep/ .components-ipfs_all .ipfs_all__icon {
    vertical-align: initial;
  }
}
</style>
