<template lang="pug">
  mixin LikeButton(isShowTotalLike="true")
    LikeButton(
      ref="likeButton"
      :version="version"
      :like-count="likeCount"
      :total-like="totalLike"
      :is-togglable="false"
      :is-max="isMaxLike"
      :is-show-max="isFlipped"
      :is-show-total-like=isShowTotalLike
      :href="popupLikeURL"
      @like="onClickLike"
      @click-stats="onClickLikeStats"
    )

  mixin SocialMediaConnect
    SocialMediaConnect(
      :username="id"
      :platforms="platforms"
      :limit="5"
    )
      block

  div(:class="rootClass")

    //- BEGIN - Version 2
    .likecoin-embed__layout
      .likecoin-embed__layout-left
        Transition(
          name="likecoin-embed__cta-badge-"
          mode="out-in"
        )
          i18n.likecoin-embed__cta-badge(
            :key="v2State"
            :path="v2CTABadgeI18nPath"
            tag="div"
          )
            br(place="br")
            a(
              place="becomeLiker"
              @click="onClickLoginButton"
            )
              | {{ $t('EmbedV2.becomeLiker') }}
            a(
              place="becomeCivicLiker"
              @click="convertLikerToCivicLiker"
            )
              | {{ $t('EmbedV2.becomeCivicLiker') }}
            a(
              place="becomePaidCivicLiker"
              @click="convertLikerToCivicLiker"
            )
              | {{ $t('EmbedV2.becomePaidCivicLiker') }}
      .likecoin-embed__layout-right.likecoin-embed__like-button-wrapper
        +LikeButton("false")

    .likecoin-embed__layout
      .likecoin-embed__layout-left
        +SocialMediaConnect
          template(#before)
            li.likecoin-embed__avatar
              LcAvatar(
                :src="avatar"
                :halo="avatarHalo"
                :is-clickable="true"
                :is-halo-clickable="true"
                size="large"
                @click="onClickAvatar"
                @click-halo="onClickAvatarHalo"
              )
      .likecoin-embed__layout-right.likecoin-embed__like-count-wrapper
        button.likecoin-embed__like-count(
          @click="onClickLikeStats"
        )
          | {{ $tc('EmbedV2.likeCountLabel', totalLike, { n: totalLike }) }}
    //- END - Version 2

</template>

<script>
import {
  checkIsMobileClient,
  requestStorageAPIAccess,
  isAndroid,
  isFacebookBrowser,
} from '~/util/client';

import CloseButtonIcon from '~/assets/like-button/close-btn.svg';

import mixin from '~/mixins/embed-button';
import LikeButton from '~/components/LikeButton';
import { logTrackerEvent } from '@/util/EventLogger';

export default {
  name: 'embed-id-button',
  layout: 'embed',
  components: {
    LikeButton,
    CloseButtonIcon,
  },
  mixins: [mixin],
  data() {
    return {
      shouldShowBackside: false,
      isUserFetched: false,
    };
  },
  computed: {
    version() {
      if (!this.$exp) return 2;
      const { name, $activeVariants } = this.$exp;
      if (name === 'like-button-v2'
        && $activeVariants.find(variant => variant.name === 'original')) return 1;
      return 2;
    },
    isMobile() {
      return checkIsMobileClient();
    },
    isFlipped() {
      return this.shouldShowBackside;
    },
    rootClass() {
      return [
        'likecoin-embed',
        'likecoin-embed--button',
        'likecoin-embed--button-v2',
        `likecoin-embed--logged-${this.isLoggedIn ? 'in' : 'out'}`,
        {
          'likecoin-embed--flipped': this.isFlipped,
          'likecoin-embed--with-halo': this.avatarHalo !== 'none',
        },
      ];
    },
    backTitle() {
      if (this.isTrialSubscriber) {
        return this.$t('Embed.back.civicLiker.trial.title');
      }
      if (this.isSubscribed) {
        return this.$t('Embed.back.civicLiker.paid.title');
      }
      return this.$t('Embed.back.civicLiker.title');
    },
    backSubtitle() {
      if (this.isSubscribed && !this.isTrialSubscriber) {
        return '';
      }
      return this.$t('Embed.back.civicLiker.subtitle');
    },
    backCTAButtonTitle() {
      if (this.isTrialSubscriber) {
        return this.$t('Embed.back.civicLiker.trial.button');
      }
      if (this.isSubscribed) {
        return this.$t('Embed.back.civicLiker.paid.button');
      }
      return this.$t('Embed.back.civicLiker.button');
    },
    v2State() {
      if (!this.isLoggedIn) {
        return 'muggle';
      }

      if (!this.isSubscribed) {
        if (this.likeCount < 1) {
          return 'liker0';
        }
        if (this.likeCount < 5) {
          return 'liker1';
        }
        return 'liker5';
      }

      if (this.isTrialSubscriber) {
        if (this.likeCount < 1) {
          return 'civicLikerTrial0';
        }
        if (this.likeCount < 5) {
          return 'civicLikerTrial1';
        }
        return 'civicLikerTrial5';
      }

      if (this.likeCount < 1) {
        return 'civicLikerPaid0';
      }
      if (this.likeCount < 5) {
        return 'civicLikerPaid1';
      }
      return 'civicLikerPaid5';
    },
    v2CTABadgeI18nPath() {
      return `EmbedV2.badge.${this.v2State}`;
    },
  },
  methods: {
    onCheckCookieSupport(isSupport) {
      if (isSupport) {
        logTrackerEvent(this, 'LikeButton', 'isCookieSupportTrue', 'isCookieSupportTrue', 1);
      } else {
        logTrackerEvent(this, 'LikeButton', 'isCookieSupportFalse', 'isCookieSupportFalse', 1);
      }
    },
    async onClickLoginButton(e) {
      logTrackerEvent(this, 'LikeButtonFlow', 'clickLoginButton', 'clickLoginButton(embed)', 1);
      if (e && typeof e.preventDefault === 'function') {
        e.preventDefault();
      }
      await this.doLogin();
    },
    async doLogin() {
      if (!this.hasCookieSupport || (isAndroid() && isFacebookBrowser())) {
        // User has not log in and 3rd party cookie is blocked
        // or: android fb iab stuck when sign in new window, use like popup
        this.popupLike();
        logTrackerEvent(this, 'LikeButtonFlow', 'popupLike', 'popupLike(embed)', 1);
        if (!(this.hasStorageAPIAccess)) {
          if (await requestStorageAPIAccess()) {
            this.hasCookieSupport = await this.getIsCookieSupport();
            await this.updateUserSignInStatus();
          }
        }
      } else {
        // User has not log in and 3rd party cookie is not blocked
        this.signIn();
        logTrackerEvent(this, 'LikeButtonFlow', 'popupSignUp', 'popupSignUp(embed)', 1);
      }
    },
    doLike() {
      if (!this.isMaxLike) {
        this.like();
        logTrackerEvent(this, 'LikeButtonFlow', 'clickLike', 'clickLike(embed)', 1);
      }

      if (this.isMaxLike) {
        this.shouldShowBackside = true;
      }
    },
    async onClickLike() {
      logTrackerEvent(this, 'LikeButtonFlow', 'clickLikeButton', 'clickLikeButton(embed)', 1);
      if (this.isLoggedIn) {
        // Case 3: User has logged in
        this.doLike();
      } else {
        await this.doLogin();
      }
    },
    onClickLikeStats() {
      this.openLikeStats();
      logTrackerEvent(this, 'LikeButtonFlow', 'clickLikeStats', 'clickLikeStats(embed)', 1);
    },
    onClickCloseButton() {
      this.shouldShowBackside = false;
    },
    onClickFrontDisplayName() {
      logTrackerEvent(
        this,
        'LikeButtonFlow',
        'clickFrontDisplayName',
        'clickFrontDisplayName(embed)',
        1,
      );
    },
    onClickBackCTAButton() {
      if (this.isSubscribed && !this.isTrialSubscriber) {
        this.superLike();
      } else {
        this.convertLikerToCivicLiker();
      }
    },
    onClickAvatar() {
      logTrackerEvent(this, 'LikeButtonFlow', 'clickAvatar', 'clickAvatar(embed)', 1);
      this.superLike();
    },
    onClickAvatarHalo() {
      logTrackerEvent(this, 'LikeButtonFlow', 'clickAvatarHalo', 'clickAvatarHalo(embed)', 1);
      this.convertLikerToCivicLiker();
    },
  },
};
</script>

<style lang="scss">
@import "~assets/css/embed";

$close-btn-width: 56;

#embed-cta-button {
  @keyframes super-like-button-shake {
    0%, 86% { transform: rotateZ(0deg); }
    88% { transform: rotateZ(2deg); }
    90% { transform: rotateZ(-2deg); }
    92% { transform: rotateZ(3deg); }
    94% { transform: rotateZ(-3deg); }
    98% { transform: rotateZ(1deg); }
    100% { transform: rotateZ(0deg); }
  }
  animation-name: super-like-button-shake;
  animation-duration: 3s;
  animation-timing-function: ease-out;
  animation-delay: -2s;
  animation-iteration-count: infinite;
  animation-fill-mode: forwards;
}

.likecoin-embed {
  perspective: 800px;

  &--with-halo {
    margin-top: normalized(24) !important;
  }

  &__badge {
    backface-visibility: hidden;
    transform-style: preserve-3d;

    .likecoin-embed--logged-out & {
      background: linear-gradient(70deg, #e6e6e6 60%, #d2f0f0, #f0e6b4);
    }

    &--front {
      .likecoin-embed--logged-out & {
        margin-right: normalized($button-width / 2 + $button-shadow-width);
      }
    }

    &--back {
      margin-right: normalized($button-width / 2 + $button-shadow-width);

      .text-content {
        padding-left: normalized(24);
      }
    }

    &-flip-- {
      &enter-active,
      &leave-active {
        transition-property: opacity, transform;
      }
      &enter-active {
        transition-timing-function: ease-out;
        transition-duration: 0.3s;
      }
      &leave-active {
        transition-timing-function: ease-in;
        transition-duration: 0.2s;
      }
      &enter {
        transform: translateZ(normalized(-100)) rotateX(-90deg);
      }
      &leave-to {
        transform: translateZ(normalized(-100)) rotateX(90deg);
      }
    }

    &__close-btn {
      position: absolute;
      top: 0;
      bottom: 0;
      left: 0;

      display: flex;
      align-items: center;
      justify-content: center;

      width: normalized($close-btn-width);

      cursor: pointer;

      transition: background-color 0.2s ease;

      border-top-left-radius: $badge-border-radius;
      border-bottom-left-radius: $badge-border-radius;

      background-color: rgba(white, 0.5);

      &:hover:not(:active) {
        background-color: rgba(white, 0.7);
      }

      > svg {
        width: normalized(28);
        height: normalized(28);

        fill: $like-green;
      }

      & + .text-content {
        padding-left: normalized($close-btn-width + 24);
      }
    }
  }

  &__badge--front,
  footer {
    margin-right: normalized($button-border-width + $button-shadow-width);
  }

  .text-content {
    &__title {
      &#{&}--amount {
        color: $like-green;

        .amount-in-usd {
          margin-left: normalized(6);

          color: $like-gray-5;

          font-size: normalized(10);
          line-height: normalized(10.5);
        }
      }

      &#{&}--civic-liker {
        font-size: normalized(24);
        line-height: normalized(24.5);
      }
    }
  }

  .login-tooltip {
    position: absolute;
    right: 0;

    margin-right: normalized(12);

    > div {
      position: relative;
    }

    &__trigger {
      display: block;

      width: normalized(16);
      height: normalized(16);

      transition-timing-function: ease;
      transition-duration: 0.25s;
      transition-property: transform, background, color;

      border: none;
      border-radius: 50%;

      @media screen and (max-width: 414px) {
        width: normalized(18);
        height: normalized(18);
      }

      &--flip- {
        &enter-active {
          transition-timing-function: ease-out;
          transition-duration: 0.15s;
        }
        &leave-active {
          transition-timing-function: ease-in;
          transition-duration: 0.05s;
        }
        &enter,
        &leave-to {
          transform: scale(0);
        }
      }

      &--open {
        color: #9b9b9b;
        background: none;

        &:hover {
          color: darken(#9b9b9b, 10);
        }
        &:active {
          color: darken(#9b9b9b, 20);
        }
      }

      &--close {
        color: white;
        background: $like-green;

        &:hover {
          color: darken(white, 20);
          background: darken($like-green, 5);
        }
        &:active {
          color: darken(white, 50);
          background: darken($like-green, 15);
        }
      }

      > div {
        width: inherit;
        height: inherit;
      }
    }

    &__bubble {
      &-wrapper {
        position: absolute;
        top: 50%;
        right: calc(100% + #{normalized(4.5)});

        transform: translateY(-50%);
      }

      position: relative;

      width: normalized(208);

      margin-right: normalized(8);
      padding: normalized(8) normalized(12);

      transform-origin: center right;

      color: #9b9b9b;

      border-radius: normalized(6);
      background-color: white;

      font-size: normalized(10);
      line-height: normalized(14);

      @media screen and (max-width: 414px) {
        width: normalized(260);

        font-size: normalized(12);
        line-height: normalized(16);
      }

      &--pop-up- {
        &enter-active {
          transition-timing-function: ease-out;
          transition-duration: 0.2s;
        }
        &leave-active {
          transition-timing-function: ease-in;
          transition-duration: 0.1s;
        }
        &enter,
        &leave-to {
          transform: scale(0);
        }
      }

      // Triangle
      &::before {
        position: absolute;
        top: 50%;
        left: 100%;

        width: 0;
        height: 0;

        content: '';

        transform: translateY(-50%);

        border-top: normalized(8) solid transparent;
        border-bottom: normalized(8) solid transparent;
        border-left: normalized(8) solid white;
      }

      a {
        text-decoration: underline;

        font-weight: 600;
      }
    }
  }

  .like-button {
    position: absolute;

    margin-top: normalized(-18);
    margin-left: normalized(44);
  }

  .social-media-connect {
    transition: opacity 0.3s ease;

    .likecoin-embed--flipped & {
      pointer-events: none;

      opacity: 0;
    }
  }
}

.likecoin-embed--button-v2.likecoin-embed {
  margin-top: normalized(30) !important;

  .likecoin-embed {
    &__layout {
      display: flex;
    }

    &__layout-left {
      width: normalized(288);
    }

    &__cta-badge {
      height: 100%;
      min-height: normalized(100);
      padding: normalized(16) normalized(20);

      border-radius: normalized(8);
      background-image: linear-gradient(56deg,#d2f0f0, #f0e6b4 65%);

      font-size: normalized(22);

      &-- {
        &enter-active,
        &leave-active {
          transition-duration: 0.4s;
          transition-property: opacity, transform;
        }
        &enter-active {
          transition-timing-function: ease-out;
        }
        &leave-active {
          transition-timing-function: ease-in;
        }
        &enter,
        &leave-to {
          opacity: 0;
        }
        &enter {
          transform: scale(0.6) rotateX(-90deg);
        }
        &leave-to {
          transform: scale(0.6) rotateX(90deg);
        }
      }
    }

    &__like-button-wrapper {
      display: flex;
      align-items: center;
      flex-shrink: 0;
    }

    &__avatar {
      margin-top: normalized(6);
      margin-right: normalized(4);
      margin-bottom: normalized(6);

      font-size: 0;

      .lc-avatar__content {
        width: normalized(48) !important;
      }
    }

    &__like-count {
      margin-left: normalized(24);
      padding: normalized(4);

      cursor: pointer;
      transition: opacity 0.25s ease;

      background: none;

      font-family: inherit;
      font-size: normalized(14);
      font-weight: 600;

      &:hover {
        opacity: 0.75;
      }

      &:active {
        opacity: 0.5;
      }

      &-wrapper {
        font-size: 0;
      }
    }
  }

  .like-button {
    position: relative;

    margin: 0;
    margin-left: normalized(6);
  }

  .social-media-connect {
    margin-right: 0;

    > div {
      margin-left: 0;
    }

    ul {
      align-items: center;
    }
  }

  &.likecoin-embed--logged-out {
    .likecoin-embed {
      &__cta-badge {
        background: linear-gradient(70deg, #e6e6e6 60%, #d2f0f0, #f0e6b4);
      }
    }
  }
}
</style>
