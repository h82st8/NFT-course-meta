<template>

  <div class="getCourseForm">
    <div class="title">
      Получить курс
    </div>

    <form @submit.prevent="onSubmit" class="form">
      <label for="email">
        <input type="email" required :class="['input', { input_notIPhone: platform !== 'iPhone' }]" id="email"
          v-model="email" placeholder="ВВЕДИТЕ свой email адрес, по нему будет предоставлен доступ к курсу" />
      </label>

      <GiftButton v-if="state === 'init' || state === 'error'">
        Получить
      </GiftButton>
    </form>

    <GiftLoadButton v-if="state === 'loading'" />

    <div v-if="state === 'success'" class="successWrapper">
      <a href="https://lms.oton.education/training/view/CHto-takoe-metavselennaya" target="_blank">
        <SentNftButton isStable>
          <span>
            Перейти на курс
          </span>
        </SentNftButton>
      </a>
      <p class="text haveKey">
        Спасибо, мы получили NFT-токен для доступа к курсу «Что такое метавселенная, и какие возможности она нам дает».
        Нажмите на кнопку «Перейти на курс», авторизуйтесь на платформе и присоединяйтесь к участникам курса в закрытом
        Telegram-чате.
      </p>
    </div>

    <div style="color: red; margin-top: 5px; text-align: center;" v-if="state === 'error'">
      Что то пошло не так. Попробуйте позже. Или обратитесь в поддержку
    </div>

  </div>
</template>

<script>
import { ref } from 'vue';
import * as Sentry from '@sentry/vue';
import { checkGift, getAcc, sendGift } from '../utils/metamask';

import GiftButton from './GiftButton.vue';
import GiftLoadButton from './GiftLoadButton.vue';
import SentNftButton from './SentNftButton.vue';

export default {
  name: 'GetCourseForm',
  components: { GiftButton, GiftLoadButton, SentNftButton },
  setup() {
    const email = ref('');
    const state = ref('init');

    const { platform } = window.navigator;

    const captureException = (error, message) => {
      console.error(error);
      Sentry.captureException(error, {
        extra: {
          message,
          platform,
        },
        user: {
          email: email.value,
        },
      });
    };

    const isGroupMember = async (userEmail) => {
      const result = await fetch(`https://lms.oton.education/oton/api/users/check?type=member&group_id=97&email=${userEmail}`);
      const json = await result.json();
      const isMember = json.data.is_member;

      console.info({ isMember });

      return isMember;
    };

    const hasNFT = async () => {
      try {
        const resultOfGifts = +(await checkGift());
        return resultOfGifts && resultOfGifts > 0;
      } catch (e) {
        captureException(e, 'hasNFT error');
        return false;
      }
    };

    const joinToGroup = async (userEmail, txid) => {
      try {
        const account = await getAcc();

        const result = await fetch(`https://lms.oton.education/oton/api/nft?address=${account}&email=${userEmail}&txid=${txid}`);

        const json = await result.json();
        const { data } = json;

        console.info({ joinData: data });

        return data.result;
      } catch (e) {
        captureException(e, 'hasNFT error');
        return false;
      }
    };

    const onSubmit = async () => {
      if (state.value === 'loading' || state.value === 'success') {
        return;
      }

      const userEmail = email.value;
      state.value = 'loading';

      let isMember;
      try {
        isMember = await isGroupMember(userEmail);
      } catch (e) {
        state.value = 'error';
        captureException(e, 'isGroupMember error');
        return;
      }

      if (isMember) {
        state.value = 'error';
        captureException(new Error('user is member'));
        return;
      }

      const isNFT = await hasNFT();
      if (!isNFT) {
        state.value = 'error';
        captureException(new Error('can not get NFT'));
        return;
      }

      let result;

      try {
        result = await sendGift('0xAC08D2F063aC3b864B0E672aa774F149624629d1');
      } catch (e) {
        state.value = 'error';
        captureException(e, 'sendGift error');
      }
      if (result) {
        const isJoined = await joinToGroup(userEmail, result.transactionHash);
        if (isJoined) {
          email.value = '';
          state.value = 'success';
        } else {
          captureException(new Error('sendGift error: can not join to group'));
          state.value = 'error';
        }
      } else {
        captureException(new Error('sendGift error: can not send gift'));
        state.value = 'error';
      }
    };

    return {
      state,
      email,
      platform,
      onSubmit,
    };
  },
};
</script>

<style lang="stylus" scoped>
.title {
  text-transform uppercase;
  font-size 22px;
  margin-bottom: 10px;
  color: #ffd600;
}
.getCourseForm {
  display flex
  flex-wrap:wrap;
  justify-content center
  margin-bottom 30px
}
.form {
  display flex
  flex-wrap:wrap;
  justify-content center
}
.successWrapper {
  display flex
  flex-direction: column
  align-items center
}
</style>
