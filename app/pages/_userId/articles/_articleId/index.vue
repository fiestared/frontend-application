<template>
  <component :is="componentName" :article="article" :topic="topicDisplayName" />
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import BlankPage from '~/components/pages/BlankPage'
import ArticleDetailV1 from '~/components/pages/ArticleDetailV1'
import ArticleDetailV2 from '~/components/pages/ArticleDetailV2'
import { htmlDecode, isV2, showEmbed } from '~/utils/article'

export default {
  components: {
    BlankPage,
    ArticleDetailV1,
    ArticleDetailV2
  },
  data() {
    return {
      componentName: 'BlankPage'
    }
  },
  async fetch({ store, params, error, redirect }) {
    try {
      const { articleId } = params
      let getArticleType = 'getArticleDetail'
      if (process.client) {
        const loggedIn = store.state.user.loggedIn
        if (loggedIn) await store.dispatch('article/setPurchasedArticleIds')
        const isCurrentUser = loggedIn && params.userId === store.state.user.currentUser.userId
        const isPurchased = loggedIn && store.state.article.purchasedArticleIds.includes(articleId)

        if (isCurrentUser) {
          getArticleType = 'getPublicArticleDetail'
        } else if (isPurchased) {
          getArticleType = 'getPurchaedArticleDetail'
        }
      }
      await Promise.all([
        store.dispatch(`article/${getArticleType}`, { articleId }),
        store.dispatch('article/getArticleSupporters', { articleId })
      ])
      if (params.userId !== store.state.article.article.user_id) {
        redirect(
          `/${store.state.article.article.user_id}/articles/${
            store.state.article.article.article_id
          }`
        )
      }
      await store.dispatch('article/getTopics')
      store.dispatch('article/setTopicDisplayName', {
        topicName: store.state.article.article.topic
      })
    } catch (e) {
      error({ statusCode: 404 })
    }
  },
  created() {
    if (isV2(this.article)) {
      this.componentName = 'ArticleDetailV2'
    } else {
      this.componentName = 'ArticleDetailV1'
    }
  },
  async mounted() {
    if (this.loggedIn) {
      const { articleId } = this.$route.params
      if (this.currentUser.phoneNumberVerified) await this.postPv({ articleId })
      await this.getIsLikedArticle({ articleId })
      await this.updateArticleCommentsByCommentIds({ articleId })

      if (this.isCurrentUser && !this.$store.state.article.isFetchedPublicArticle) {
        await this.$store.dispatch('article/getPublicArticleDetail', { articleId })
        this.$store.dispatch('article/setIsFetchedPublicArticle', { isFetched: true })
        const paywallLine = document.querySelector('.paywall-line')
        if (paywallLine) {
          paywallLine.innerHTML = `これより上のエリアが<span class="br" />無料で表示されます`
        }
        showEmbed()
        return
      }

      await this.$store.dispatch('article/setPurchasedArticleIds')
      const isPurchased =
        this.loggedIn && this.$store.state.article.purchasedArticleIds.includes(articleId)
      if (
        isPurchased &&
        !this.isCurrentUser &&
        !this.$store.state.article.isFetchedPurchasedArticle
      ) {
        await this.$store.dispatch('article/getPurchaedArticleDetail', { articleId })
        const paywallLine = document.querySelector('.paywall-line')
        if (paywallLine) paywallLine.remove()
        showEmbed()
      }
    } else {
      this.setIsLikedArticle({ liked: false })
    }
  },
  computed: {
    isCurrentUser() {
      return this.loggedIn && this.$route.params.userId === this.currentUser.userId
    },
    ...mapGetters('user', ['loggedIn', 'currentUser']),
    ...mapGetters('article', ['article', 'topicDisplayName'])
  },
  methods: {
    ...mapActions('article', [
      'postPv',
      'getIsLikedArticle',
      'setIsLikedArticle',
      'updateArticleCommentsByCommentIds'
    ])
  },
  head() {
    const { article } = this.$store.state.article
    const decodedArticleTitle = htmlDecode(article.title)
    const decodedArticleOverview = htmlDecode(article.overview)
    const eyeCatchUrl = article.eye_catch_url ? `${article.eye_catch_url}?d=1200x630` : null

    return {
      title: decodedArticleTitle,
      link: [
        {
          rel: 'stylesheet',
          href:
            'https://cdnjs.cloudflare.com/ajax/libs/medium-editor-insert-plugin/2.4.1/css/medium-editor-insert-plugin.min.css'
        }
      ],
      meta: [
        {
          hid: `og:title`,
          property: 'og:title',
          content: `${decodedArticleTitle} | ALIS`
        },
        {
          hid: `og:description`,
          property: 'og:description',
          content: decodedArticleOverview
        },
        {
          hid: `og:type`,
          property: 'og:type',
          content: 'article'
        },
        {
          hid: `og:image`,
          property: 'og:image',
          content:
            eyeCatchUrl || `https://${process.env.ALIS_APP_DOMAIN}/d/nuxt/dist/OGP_1200×630.png`
        },
        {
          hid: 'twitter:image',
          name: 'twitter:image',
          content:
            eyeCatchUrl || `https://${process.env.ALIS_APP_DOMAIN}/d/nuxt/dist/OGP_1200×630.png`
        }
      ]
    }
  }
}
</script>
