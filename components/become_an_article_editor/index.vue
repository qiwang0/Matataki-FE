<template>
  <div
    v-if="(isTokenArticle || isPriceArticle) && !isProduct"
    v-loading="lockLoading"
    class="lock"
  >
    <div class="lock-left">
      <img
        v-if="!(hasPaied && hasPaiedRead)"
        class="lock-img"
        src="@/assets/img/lock.png"
        alt="lock"
      >
      <img
        v-else
        class="lock-img"
        src="@/assets/img/unlock.png"
        alt="lock"
      >
    </div>
    <div class="lock-info">
      <h3 class="lock-info-title">
        {{ !(hasPaied && hasPaiedRead) ? `${$t('unlock-edit-permissions', [ unlockText ])}` : `${$t('unlock-edit-permission', [ unlockText ])}` }}
      </h3>
      <h5 class="lock-info-subtitle">
        {{ !(hasPaied && hasPaiedRead) ? $t('you-need-to-meet-the-following-unlock-conditions') : $t('you-have-fulfilled-the-following-unlock-conditions') }}
        <el-tooltip
          effect="dark"
          :content="$t('you-can-edit-the-article-after-all-the-conditions-are-met')"
          placement="top-start"
        >
          <svg-icon
            icon-class="anser"
            class="prompt-svg"
          />
        </el-tooltip>
      </h5>
      <p
        v-if="!isMe(article.uid)"
        class="lock-info-des"
      >
        <ul>
          <li
            v-if="isTollRead"
            class="fl"
          >
            <div class="fl price">
              {{ $t('need-to-unlock-the-permission-to-read-this-article-first') }}
              <svg-icon
                icon-class="read"
                class="avatar-read"
              />
            </div>
            <el-tooltip
              effect="dark"
              :content="$t('this-article-has-reading-restrictions-if-you-need-to-edit-you-must-obtain-reading-permissions')"
              placement="left"
            >
              <svg-icon icon-class="anser" />
            </el-tooltip>
          </li>
          <li
            v-if="isPriceArticle"
            class="fl"
          >
            <div class="fl price">
              {{ $t('pay') }}
              <span class="amount">{{ getArticlePrice }}</span>
              <svg-icon
                icon-class="currency"
                class="avatar-cny"
              />
              {{ $t('mttk-points') }}
            </div>
            <el-tooltip
              effect="dark"
              :content="$t('articles-unlocked-by-payment-can-be-viewed-permanently-in-the-Purchase-History')"
              placement="left"
            >
              <svg-icon icon-class="anser" />
            </el-tooltip>
          </li>
          <li
            v-if="isTokenArticle"
            class="fl"
          >
            <div class="fl price">
              {{ $t('hold') }}
              <span class="amount">{{ needTokenAmount }}</span>
              <router-link
                :to="{name: 'token-id', params:{ id:needTokenId }}"
                target="_blank"
                class="fl"
              >
                <avatar
                  :size="'16px'"
                  :src="needTokenLogo"
                  class="avatar-token"
                />
                {{ needTokenSymbol }}（{{ needTokenName }}）
              </router-link>
            </div>
            <!-- 不显示 - 号 -->
            <span> {{ !tokenHasPaied ? $t('still-need-to-hold') : $t('already-held') }}{{ isLogined ? differenceToken.slice(1) : needTokenAmount }} {{ needTokenSymbol }}</span>
          </li>
        </ul>
      </p>
      <p
        v-else
        class="lock-info-des"
      >
        {{ $t('self-publish-article') }}
      </p>
      <div
        v-if="!hasPaied"
        class="lock-bottom notice-creator-container"
      >
        <div class="btn-ccc">
          <!-- <span class="lock-bottom-total">总计约{{ totalCny }}{{ $t('mttk-points') }}</span> -->
          <el-tooltip
            effect="dark"
            :content="$t('click-and-pay-to-unlock-editing-permissions-if-there-are-reading-restrictions-please-unlock-purchase-the-full-text-first')"
            placement="top-end"
          >
            <el-button
              type="primary"
              size="small"
              @click="wxpayEdit"
            >
              {{ $t('one-key') }}{{ unlockText }}
            </el-button>
          </el-tooltip>
        </div>
      </div>
      <div
        v-else
        class="lock-bottom"
      >
        <span
          v-if="!hasPaiedRead"
          class="lock-bottom-total"
        >{{ $t('this-article-has-reading-restrictions-if-you-need-to-edit-you-must-obtain-reading-permissions') }}</span>
        <el-button
          type="primary"
          size="small"
          :disabled="!hasPaiedRead"
          @click="edit"
        >
          {{ $t('edit-article') }}
        </el-button>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import avatar from '@/components/avatar/index.vue'
import { precision } from '@/utils/precisionConversion'
import utils from '@/utils/utils'

export default {
  name: 'ArticleCard',
  components: {
    avatar,
  },
  props: {
    // 文章数据
    article: {
      type: Object,
      required: true
    },
    // 是否已解锁（购买）编辑权限
    hasPaied: {
      type: Boolean,
      default: false
    },
    // 是否已解锁（购买）编辑权限
    tokenHasPaied: {
      type: Boolean,
      default: false
    },
    differenceToken: {
      type: String,
      default: '0'
    },
    form: {
      type: Object,
      required: true
    },
    inputAmountError: {
      type: String,
      default: ''
    },
    lockLoading: {
      type: Boolean,
      default: false
    },
    // 是收费文章
    isTollRead: {
      type: Boolean,
      default: false
    },
    // 已解锁阅读权限
    hasPaiedRead: {
      type: Boolean,
      default: true
    },
    editTokenExs: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      isProduct: false,
    }
  },
  computed: {
    ...mapGetters(['currentUserInfo', 'isLogined', 'isMe']),
    unlockText() {
      if (this.isPriceArticle) {
        return '购买'
      }
      return '解锁'
    },
    // 是否是持通证文章
    isTokenArticle() {
      return (this.article.editTokens && this.article.editTokens.length !== 0)
    },
    // 是否是付费文章
    isPriceArticle() {
      return (this.article.editPrices && this.article.editPrices.length !== 0)
    },
    // 价格
    getArticlePrice() {
      if (this.isPriceArticle && this.article.editPrices[0]) {
        const ad = this.article.editPrices[0]
        return this.$utils.fromDecimal(ad.price)
      } else {
        return 0
      }
    },
    // 需要多少Fan票
    needTokenAmount() {
      if (this.article.editTokens.length !== 0) {
        return precision(this.article.editTokens[0].amount, 'CNY', this.article.editTokens[0].decimals)
      } else return 0
    },
    // Fan票ID
    needTokenId() {
      if (this.article.editTokens.length !== 0) {
        return this.article.editTokens[0].id
      } else return -1
    },
    // 需要多少Fan票代号
    needTokenSymbol() {
      if (this.article.editTokens.length !== 0) {
        return this.article.editTokens[0].symbol
      } else return ''
    },
    // 需要多少Fan票名称
    needTokenName() {
      if (this.article.editTokens.length !== 0) {
        return this.article.editTokens[0].name
      } else return ''
    },
    // 需要多少Fan票LOGO
    needTokenLogo() {
      if (this.article.editTokens.length !== 0) {
        return this.$ossProcess(this.article.editTokens[0].logo)
      } else return ''
    },
    // 总价
    totalCny() {
      let result = 0
      if (this.isTokenArticle) {
        result += parseFloat(this.form.input || 0)
      }
      return utils.up2points(result + this.getArticlePrice)
    }
  },
  methods: {
    // 购买编辑权限
    wxpayEdit() {
      if (!this.isLogined) {
        this.$store.commit('setLoginModal', true)
        return false
      }
      this.$emit('createOrder', { nt: this.isTokenArticle && !this.tokenHasPaied })
    },
    edit() {
      if(this.article && this.article.hash) {
        this.$router.push({
          name: 'publish-type-id',
          params: { type: 'edit', id: this.article.id },
          query: { hash: this.article.hash }
        })
      }
      else this.$message.error('无法获取文章Hash')
    }
  }
}
</script>

<style lang="less" scoped>
.lock {
  background: #fff;
  display: flex;
  &-left {
    align-self: flex-start;
    margin-right: 10px;
  }
  .lock-img {
    width: 46px;
  }
  &-info {
    width: 100%;
    &-title {
      font-size: 22px;
      color: #000000;
      padding: 0;
      margin: 4px 0;
    }
    &-subtitle {
      font-size: 16px;
      color: #B2B2B2;
      padding: 0;
      margin: 8px 0;
      font-weight: 400;
    }
  }
  &-info-des {
    font-size: 16px;
    font-weight: 400;
    color: #000000;
    padding: 0;
    margin: 0;
    display: flex;
    justify-content: space-between;
    align-items: center;
    span {
      color:rgba(178,178,178,1);
      margin-left: 6px;
      &.amount {
        color: #000000;
        margin-left: 5;
        min-width: 36px;
      }
    }
    a {
      color:#542DE0;
    }
    .price {
      flex: 1;
    }
  }
  &-bottom {
    display: flex;
    justify-content: flex-end;
    align-content: center;
    border-top: 1px solid #DBDBDB;
    margin-top: 10px;
    padding-top: 10px;
    align-items:center;
    &-total {
      font-size: 14px;
      color: #B2B2B2;
      margin-right: 20px;
      display: flex;
      align-items: center;
    }
  }
  ul {
    list-style-type: none;
    padding: 0;
    margin: 10px 0 0 0;
    width: 100%;
    li {
      margin-bottom: 10px;
    }
  }
  .help-icon {
    color:#DBDBDB;
  }
}

.avatar {
  &-token {
    margin: 0 5px;
  }
  &-cny {
    margin: 0 6px;
  }
  &-read {
    margin: 0 6px;
    color: #848484;
  }
}

.prompt-svg-token {
  padding-left: 8px;
}
.notice-creator-container {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: center;
  .notice-creator {
    margin-top: 10px;
    .warn-tip {
      color: #FB6877!important;
      font-size: 12px;
    }
  }
}
.btn-ccc {
  display: flex;
}
</style>
