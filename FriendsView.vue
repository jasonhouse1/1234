<script setup>
import {
    onMounted,
    onBeforeUnmount,
    ref,
    watchEffect,
    watch,
    defineComponent,
    inject,
    computed,
    toRaw
  } from 'vue';

  import { useRoute, useRouter } from 'vue-router';
  import axios from 'axios';
  import $ from "jquery";

  // 多國語言
  import { useI18n } from 'vue-i18n';
  const { t, tm, locale } = useI18n();

  const router = useRouter();
  const route = useRoute();
  const userInfoObj = ref(JSON.parse(localStorage.getItem("userInfo")));
  const isAuthenticated = ref(true);

  const friendsObj = ref({
        'mgm_code': '',
        'mgm_invited_count': 0,
        'mgm_finish_count': 0,
        'mgm_earn_money': 0,
        });

  function calculateMgmEarnMoney(finishCount) {
    console.log(finishCount);
    if (finishCount <= 0) return 0;
    if (finishCount == 1) return 50;
    if (finishCount == 2) return 100;
    if (finishCount == 3) return 200;
    if (finishCount == 4) return 300;
    return 400; 
  }

  
  // 對話框
  const commonDialog = ref(false);
  const commonDialogText = ref("");

  const IPicon = {
    ip: new URL(`/images/index/IPicon.png`, import.meta.url).href,
  };

  const startBtnLoading = ref(false);

  const robotText = inject('$robotText');

  onMounted(() => {
    robotText.value = '';

    if (!localStorage.getItem("accounts")) {
      isAuthenticated.value = false;
    } else {

      handleLoadFriendsInfo();

      emailTagInit();
    }

    updateMainClass(route.path);
  
  });

  const copyLink = async () => {
    if (friendsObj.value.mgm_code === '') {
      // console.log('mgm_code 為空，未執行複製')
      return;
    }

    try {
      await navigator.clipboard.writeText('https://socialsupermanagers.com/ssm/login/register/'+friendsObj.value.mgm_code);
      alert('連結已複製！');
    } catch (err) {
      alert('複製失敗');
    }
  };

  const shareTo = (platform) => {

    if (friendsObj.value.mgm_code === '') {
      // console.log('mgm_code 為空，未執行複製')
      return;
    }

    const encodedUrl = encodeURIComponent('推薦你一起體驗「科學小編俠」，來免費體驗高階版 1 個月！ https://socialsupermanagers.com/ssm/login/register/'+friendsObj.value.mgm_code);
    const encodedUrlFromFb = encodeURIComponent('https://socialsupermanagers.com/ssm/login/register/'+friendsObj.value.mgm_code);
    let shareUrl = '';

    switch (platform) {
      case 'line':
        shareUrl = `https://line.me/R/msg/text/?${encodedUrl}`;
        break;
      case 'facebook':
        shareUrl = `https://www.facebook.com/sharer/sharer.php?u=${encodedUrlFromFb}`;
        break;
      case 'twitter':
        shareUrl = `https://twitter.com/intent/tweet?url=${encodedUrl}`;
        break;
      case 'whatsapp':
        shareUrl = `https://api.whatsapp.com/send?text=${encodedUrl}`;
        break;
    }

    window.open(shareUrl, '_blank');
  };

  const emailTagInit = () => {
    const emailInput = document.getElementById("emailInput");
    const emailInputCover = document.getElementById("emailInputCover");
    const emailHidden = document.getElementById("emailHiddenInput");
    const sendBtn = document.getElementById("sendBtn");

    const isValidEmail = (email) => {
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    };

    const createEmailTag = (email) => {
      const div = document.createElement("div");
      div.classList.add("email-tag");

      if (!isValidEmail(email)) {
        div.classList.add("invalid");
      }

      const text = document.createTextNode(email);

      const closeBtn = document.createElement("i");
      closeBtn.classList.add("close_btn");

      // 加入刪除事件
      closeBtn.addEventListener("click", () => {
        div.remove();
      });

      div.appendChild(text);
      div.appendChild(closeBtn);
      return div;
    };

    emailInput.addEventListener("keydown", (e) => {
      if (e.key === " " || e.key === "Enter") {
        e.preventDefault();
        const text = emailInput.textContent.trim();
        if (text.length === 0) return;

        const emails = text.split(/\s+/);
        emails.forEach((email) => {
          if (isValidEmail(email)) {
            const tag = createEmailTag(email);
            emailInputCover.appendChild(tag); // append only valid email
          } else {
            // Optional: 顯示錯誤提醒
            alert(`無效的Email`);
          }
        });

        emailInput.textContent = "";
      }
    });

    sendBtn.addEventListener("click", async () => {
      const tags = emailInputCover.querySelectorAll(".email-tag");

      // 抓每個 tag 中 email 的內容
      const emailList = Array.from(tags)
        .map((tag) => tag.firstChild?.textContent?.trim() || "")
        .filter((email) => isValidEmail(email));

      if (emailList.length === 0) {
        alert("需輸入Email並按下空白鍵才能發送");
        return;
      }

      // 確認是否送出
      const confirmed = confirm("即將發送至以下 Email：\n" + emailList.join(", "));
      if (!confirmed) return;

      // 寫入隱藏 input，用於表單送出
      emailHidden.value = emailList.join(",");

      
      let response;
      try {
        response = await axios.post(
          "https://socialsupermanagers.com/ssm/lib/subscribe/email_mgm_send.php",
          {
            method: "sendEmail",
            user: userInfoObj.value["sid"],
            name: userInfoObj.value["name"],
            emailArr: emailHidden.value,
            code: friendsObj.value.mgm_code,
          },
          { headers: { "Content-Type": "multipart/form-data" } }
        );
      } catch (error) {
        alert("發送失敗，請稍後再試！");
        return;
      }

      const res = response.data;

      //  寄送成功後清空所有 email tag
      emailInputCover.innerHTML = "";

      //  可選提示
      alert("Email 發送成功！");

    });
  };


  router.afterEach((to) => {
    updateMainClass(to.path); // 切換路由後根據「新的路由」判斷
  });

  function updateMainClass(path) {
    if (path.match("/friends")) {
      $('main').addClass('friends_page');
    } else {
      $('main').removeClass('friends_page');
    }
  }

  const handleLoadFriendsInfo = async () => {

    let response;
    try {
      response = await axios.post(
        "https://socialsupermanagers.com/ssm/api/friends/apiFriends.php",
        {
          method: "getFriendsInfo",
          user: userInfoObj.value["sid"],
        },
        { headers: { "Content-Type": "multipart/form-data" } }
      );
    } catch (error) {
      // handleError(error);
    }

    const res = response.data;
    friendsObj.value.mgm_code = res.mgm_code;
    friendsObj.value.mgm_invited_count = res.mgm_invited_count;
    friendsObj.value.mgm_finish_count = res.mgm_finish_count;
    friendsObj.value.mgm_earn_money = calculateMgmEarnMoney(friendsObj.value.mgm_finish_count);

    //console.log(res);
  };

  const goToSectionBtn = (id) => {
    const el = document.getElementById(id);
    if (!el) return;

    // 計算元素距離頁面頂部的位置
    const offsetTop = el.getBoundingClientRect().top + window.scrollY;

    // 加上偏移量（例如往下 70px）
    const finalScroll = offsetTop - 70;

    // 平滑捲動
    setTimeout(() => {
      window.scrollTo({
        top: finalScroll,
        behavior: 'smooth'
      });
    }, 100);
    /*
    if(getId.id === 'info') {
      gtag('event', '[小編俠] 首頁-點擊立即購買');
      fbq('track', '[intro]click buy now', {
      content_name: '[intro] 首頁-點擊立即購買',
      action: 'click'
    });
      dataLayer.push({
        'event': 'gtm_click', 
        'click': '[小編俠] 首頁-點擊立即購買'
      });
      console.log('點擊立即購買');
    }
    */
  };

</script>

<template>

   <div class="fixed-right-btn d-flex flex-column">
    <v-btn class="mb-3" @click="commonDialog=true;commonDialogText='即將開放';"><v-img :src="IPicon.ip"></v-img></v-btn>
    <v-btn class="btn_style" @click="router.push('/intro')" v-html="t('intro.side.try_btn')"></v-btn>
  </div>
  <div class="friends_page_container">
    <v-overlay v-model="startBtnLoading" class="align-center justify-center" persistent>
      <v-progress-circular indeterminate color="white" size="64"></v-progress-circular>
    </v-overlay>
    <!-- banner -->
    <div class="friends_page_banner_container">
      <div class="friends_page_banner">
        <div class="left_container">
          <div class="top_title">{{ t('friends.banner.topTitle') }} ｜ {{ t('friends.banner.topTitle_2') }}</div>
          <div class="middle_title">
            <div class="title_1_text">{{ t('friends.banner.title1') }}</div>
            <div class="title_2_text">{{ t('friends.banner.title2') }} </div>
            <div class="title_2_text_light_mode_">{{ t('friends.banner.title2Light') }}</div>
          </div>
          <div class="bottom_comment">{{ t('friends.banner.comment') }}</div>
          <div class="start_btn_container">
            <v-btn class="start_btn" @click="goToSectionBtn('HowToInvite')"> {{ t('friends.banner.btnInvite') }}</v-btn>
            <div class="comment_" @click="goToSectionBtn('InviteStep')">{{ t('friends.banner.btnDetail') }}</div>
          </div>
        </div>
        <div class="right_container">
          <div class="kerker_img"></div>
          <div class="semiconductor_img"></div>
          <div class="coin_img_1"></div>
          <div class="coin_img_2"></div>
          <div class="coin_img_3"></div>
          <div class="coin_img_4"></div>
          <div class="coin_img_5"></div>
          <div class="coin_img_6"></div>
        </div>
      </div>
      <div class="friends_page_banner_bg_img"></div>
      <div class="right_stars_bg"></div>
    </div>
    

    <!-- 主頁面 -->
    <div class="friends_page_content">
      <!-- 選擇以何形式發送邀請？ -->
      <div class="card_content card_1" id="HowToInvite">
        <div class="title_container">
          <h2>{{ t('friends.invite.title') }}</h2>
        </div>
        <div class="boxshadow_container invite_link_container">
          <div class="invite_link_block">
            <div class="title_">{{ t('friends.invite.shareLink') }}<span class="small_" v-if="!isAuthenticated"> ({{
                t('common.login_required') }} <a href="./login">{{ t('common.login') }}</a>)</span></div>
            <div class="share_inout_container">
              <input type="text" name="shareInput" class="share_input" :placeholder="t('friends.field.share_input_placeholder')" readonly
                :value="(friendsObj.mgm_code==''?'':'https://socialsupermanagers.com/ssm/login/register/'+friendsObj.mgm_code)" />
              <v-btn id="shareInputBtn" class="share_input_btn" @click="copyLink"> {{ t('friends.invite.btnCopy') }}</v-btn>
            </div>
            <div class="social_share_list_container">
              <div class="social_share_title">{{ t('friends.invite.shareTo') }}</div>
              <v-btn class="social_share_container" @click="shareTo('line')"><i class="line"></i></v-btn>
              <v-btn class="social_share_container" @click="shareTo('facebook')"><i class="facebook"></i></v-btn>
              <v-btn class="social_share_container" @click="shareTo('twitter')"><i class="twitter"></i></v-btn>
              <v-btn class="social_share_container" @click="shareTo('whatsapp')"><i class="whatsapp"></i></v-btn>
            </div>
          </div>
          <div class="invite_link_block">
            <div class="title_"> {{ t('friends.invite.shareEmail') }} <span class="small_" v-if="!isAuthenticated"> ({{
                t('common.login_required') }} <a href="./login">{{ t('common.login') }}</a>)</span></div>
            <div class="share_inout_container">
              <div id="emailInput" class="share_email_input" contenteditable="true"
                :placeholder="t('friends.invite.placeholderEmail')"></div>
              <div class="email_input_cover" id="emailInputCover"></div>
              <input type="hidden" id="emailHiddenInput" name="email" />
              <button id="sendBtn" class="share_input_btn">{{ t('friends.invite.btnSend') }}</button>
            </div>
            <div class="comment_container">
              {{ t('friends.invite.commentEmail') }}
            </div>
          </div>

        </div>
      </div>
      <!-- 三步驟完成推薦任務 -->
      <div class="card_content card_2" id="InviteStep">
        <div class="title_container">
          <h2>{{ t('friends.steps.title') }}</h2>
        </div>
        <div class="mission_list_container">
          <div class="boxshadow_container mission_container">
            <div class="step_count_bg"></div>
            <div class="mission_content">
              <div class="title_">{{ t('friends.steps.step1Title') }}</div>
              <div class="content_">{{ t('friends.steps.step1Content') }}</div>
              <div class="bottom_icon"><i class=""></i></div>
            </div>
          </div>
          <div class="boxshadow_container mission_container">
            <div class="step_count_bg"></div>
            <div class="mission_content">
              <div class="title_">{{ t('friends.steps.step2Title') }}</div>
              <div class="content_">{{ t('friends.steps.step2Content') }}</div>
              <div class="bottom_icon"><i class=""></i></div>
            </div>
          </div>
          <div class="boxshadow_container mission_container">
            <div class="step_count_bg"></div>
            <div class="mission_content">
              <div class="title_">{{ t('friends.steps.step3Title') }}</div>
              <div class="content_">{{ t('friends.steps.step3Content') }}</div>
              <div class="bottom_icon"><i class=""></i></div>
            </div>
          </div>
        </div>
      </div>
      <!-- 推薦進度條 -->
      <div class="card_content card_3">
        <div class="title_container">
          <h2>{{ t('friends.progress.title') }}</h2>
        </div>
        <!-- 推薦進度條 -->
        <div class="boxshadow_container progress_bar_container">
          <div class="bar_container">
            <div class="bar_proccess" :style="{ width: (130 * (friendsObj.mgm_finish_count-1)) + 'px' }"></div>
          </div>
          <div class="invite_text_float">{{ t('friends.progress.invite') }}</div>
          <div class="earn_coin_text_float">{{ t('friends.progress.reward') }}</div>

          <div class="step" v-for="(friend, index) in tm('friends.progress.friends')" :key="index">
            <div class="circle" :class="{ crown: index === 4 }"></div>
            <div class="label">{{ friend }}</div>
            <div class="coin_">
              NT${{ (index + 1) * 50 }} <i v-if="index === 4" class="crown_"></i>
            </div>
          </div>
        </div>

        <!-- 統計卡片 -->
        <div class="stat_card_container">
          <div class="boxshadow_container stat_card">
            <div class="stat_number">{{ friendsObj.mgm_invited_count }}</div>
            <div class="stat_label">{{ t('friends.stat.invited') }}</div>
          </div>
          <div class="boxshadow_container stat_card">
            <div class="stat_number">{{ friendsObj.mgm_finish_count }}</div>
            <div class="stat_label">{{ t('friends.stat.completed') }}</div>
          </div>
          <div class="boxshadow_container stat_card">
            <div class="stat_number">{{ friendsObj.mgm_earn_money }}</div>
            <div class="stat_label">{{ t('friends.stat.credits') }} <i class="dollar_coin"></i></div>
            <div class="stat_icon">
              <div class="btn_container_" v-if="friendsObj.mgm_earn_money > 0">
                <v-btn class="redeem_btn" to="/member/level_intro">兌換</v-btn>
              </div>
            </div>
          </div>
        </div>

        <!-- 常見問題 -->
        <div class="card_content card_4">
          <div class="title_container">
            <h2>{{ t('friends.faqTitle') }}</h2>
          </div>
          <div class="boxshadow_container faq_container">
            <div class="faq_content">
              <v-expansion-panels>
                <!-- Q1 -->
                <v-expansion-panel elevation="0">
                  <v-expansion-panel-title collapse-icon="mdi-minus" expand-icon="mdi-plus">
                    <h4>Q1: {{ t('friends.faq.q1') }}</h4>
                  </v-expansion-panel-title>
                  <v-expansion-panel-text>
                    <h4 class="fw-light">A: {{ t('friends.faq.a1') }}</h4>
                  </v-expansion-panel-text>
                </v-expansion-panel>

                <!-- Q2 -->
                <v-expansion-panel elevation="0">
                  <v-expansion-panel-title collapse-icon="mdi-minus" expand-icon="mdi-plus">
                    <h4>Q2: {{ t('friends.faq.q2') }}</h4>
                  </v-expansion-panel-title>
                  <v-expansion-panel-text>
                    <h4 class="fw-light">A: {{ t('friends.faq.a2') }}</h4>
                  </v-expansion-panel-text>
                </v-expansion-panel>

                <!-- Q3 -->
                <v-expansion-panel elevation="0">
                  <v-expansion-panel-title collapse-icon="mdi-minus" expand-icon="mdi-plus">
                    <h4>Q3: {{ t('friends.faq.q3') }}</h4>
                  </v-expansion-panel-title>
                  <v-expansion-panel-text>
                    <h4 class="fw-light">
                      A: <span style="color: red;">{{ t('friends.faq.highlightNewUser') }}</span>
                      {{ t('friends.faq.a3') }}
                    </h4>
                  </v-expansion-panel-text>
                </v-expansion-panel>

                <!-- Q4 -->
                <v-expansion-panel elevation="0">
                  <v-expansion-panel-title collapse-icon="mdi-minus" expand-icon="mdi-plus">
                    <h4>Q4: {{ t('friends.faq.q4') }}</h4>
                  </v-expansion-panel-title>
                  <v-expansion-panel-text>
                    <h4 class="fw-light">A: {{ t('friends.faq.a4') }}</h4>
                  </v-expansion-panel-text>
                </v-expansion-panel>

                <!-- Q5 -->
                <v-expansion-panel elevation="0">
                  <v-expansion-panel-title collapse-icon="mdi-minus" expand-icon="mdi-plus">
                    <h4>Q5: {{ t('friends.faq.q5') }}</h4>
                  </v-expansion-panel-title>
                  <v-expansion-panel-text>
                    <h4 class="fw-light">A: {{ t('friends.faq.a5') }}</h4>
                  </v-expansion-panel-text>
                </v-expansion-panel>
              </v-expansion-panels>
            </div>
          </div>
        </div>

        <!-- 免責聲明與注意事項 -->
        <div class="card_content card_5">
          <div class="card_5_bg"></div>
          <div class="title_container">
            <h2>{{ t('friends.disclaimer.title') }}</h2>
          </div>
          <div class="boxshadow_container friends_container">
            <ul class="friends_content">
              <li v-for="(item, index) in tm('friends.disclaimer.list')" :key="index">
                {{ item }}
              </li>
            </ul>
          </div>
        </div>

        <!-- footer -->
        <div class="footer_container">
          <div class="footer_content">
            © <a href="https://www.aretedigitalsocial.com/" target=_blank>亞瑞特數位社群行銷有限公司</a> 版權所有<br>
            Copyright © ARETE INC. All Rights Reserved.
          </div>
        </div>
      </div>

    </div>
    <div class="">
      <v-dialog class="level_intro_modal_container_2" v-model="commonDialog" width="auto">
        <v-card>
          <v-card-text>
            <div class="dialog_container">
              <div class="close_btn_container">
                <v-btn class="close_btn" @click="commonDialog = false"><span class="mdi mdi-close"></span></v-btn>
              </div>
              <div class="dialog_content">
                {{ commonDialogText }}
                <br />
                <div class="button_container">
                  <v-btn @click="commonDialog = false">確定</v-btn>
                </div>
              </div>
            </div>
          </v-card-text>
        </v-card>
      </v-dialog>
    </div>
    </div>
</template>

<style lang="scss">

  
  .main_container .content_container main.friends_page {
    max-width: 100%;
    width: 100%;
  }

  .friends_page_container {
    .email_input_cover {
      display: flex;
      flex-direction: column;
      float: left;
      .email-tag {
        margin: 2px 2px;
        width: auto;
        background: #e4efff;
        color: #666;
        padding: 4px 12px;
        font-size: 14px;
        border-radius: 17px;
        display: flex;
        align-items: center;
        justify-content: center;
        &:first-child {
          margin-top: 10px;
        }
        .close_btn {
          background-image: url(/images/close_btn.svg);
          background-position: center;
          background-size: contain;
          background-repeat: no-repeat;
          display: inline-block;
          width: 17px;
          height: 17px;
          cursor: pointer;
          margin-left: 5px;
        }
      }
    }
  }
</style>

<style lang="scss" scoped>

  :deep(.v-btn__content) {
    font-family: "Noto Sans TC", sans-serif;
  }

  :deep(.v-btn.v-btn--density-default) {
    width: 224px;
    height: 58px;
    background: rgba(0, 216, 205, 1);
    border-radius: 50px;
    color: #fff;
    font-size: 18px;
    font-family: "Noto Sans TC", sans-serif;
  }

  .friends_page_container {
    
    .friends_page_banner_container {
      background: linear-gradient(0deg, #D5EDFF, #D5EDFF);
      position: relative;
      z-index: 0;
      margin-top: -30px;
      .friends_page_banner {
        z-index: 2;
        max-width: 900px;
        transform: scale(1.2);
        height: 404px;
        width: 95%;
        margin: 0 auto;
        padding: 20px 0px;
        display: flex;
        position: relative;
        @media screen and (min-width: 768px) and (max-width: 1200px) {
          transform: scale(1);
        }
        @media screen and (max-width: 767px) {
          flex-direction: column;
          max-width: 400px;
          transform: scale(1);
          height: auto;
        }
        .left_container {
          width: 50%;
          color: #5B5B5B;
          @media screen and (min-width: 768px) and (max-width: 950px) {
            width: 440px
          }
          @media screen and (max-width: 767px) {
            width: 100%;
          }
          .top_title {
            color: #638FF9;
            font-size: 20px;
            margin-top: 60px;
            font-weight: 500;
            letter-spacing: 1px;
            @media screen and (max-width: 767px) {
              text-align: center;
              font-size: 18px;
              margin-top: 100px;
            }
          }
          .middle_title {
            margin-top: 0px;
            color: #3150F3;
            @media screen and (max-width: 767px) {
              display: flex;
              flex-direction: row;
              flex-wrap: wrap;
              justify-content: center;
            }
            .title_1_text {
              font-size: 38px;
              font-weight: 800;
              letter-spacing: 2px;
            }
            .title_2_text {
              font-size: 38px;
              font-weight: 800;
              letter-spacing: 2px;
              margin-top: -5px;
              margin-left: 43px;
              float: left;
              @media screen and (max-width: 767px) {
                margin-left: 0;
                margin-top: 0;
                /* width: auto; */
                float: initial;
              }
            }
            .title_2_text_light_mode_ {
              color: #00D8CD;
              display: inline-block;
              transform: skew(-10deg, 0deg);
              letter-spacing: -1px;
              font-size: 38px;
              font-weight: 800;
              /* float: left; */
              margin-left: 5px;
              margin-top: -5px;
            }
          }
          .bottom_comment {
            font-size: 19px;
            letter-spacing: 1px;
            @media screen and (max-width: 767px) {
              font-size: 14px;
              text-align: center;
            }
          }
          .start_btn_container {
            display: flex;
            flex-direction: column;
            align-content: center;
            align-items: center;
            width: 300px;
            margin-top: 5px;
            @media screen and (max-width: 767px) {
              width: 100%;
            }
            .start_btn {
              padding: 9px 45px;
              /* gap: 10px; */
              /* position: absolute; */
              /* width: 224px; */
              height: 51px;
              /* left: 88px; */
              /* top: 265px; */
              background: #00D8CD;
              box-shadow: 0px 0px 25px rgba(0, 0, 0, 0.25);
              border-radius: 50px;
              color: #fff;
              font-size: 20px;
              margin-top: 15px;
            }
            .comment_ {
              font-size: 12px;
              letter-spacing: 1px;
              margin-top: 15px;
              cursor: pointer;
            }
          }
        }
        .right_container {
          width: 50%;
          margin-left: 0px;
          padding-bottom: 20px;
          @media screen and (max-width: 767px) {
            width: 100%;
            height: 300px;
            margin: 0;
          }
          .kerker_img {
            background-image: url(/images/friends_banner_img.png);
            background-size: auto 288px;
            background-position: center bottom;
            background-repeat: no-repeat;
            /* width: calc(50% + 150px); */
            height: 100%;
            @media screen and (min-width: 768px) and (max-width: 950px) {
              background-size: auto 288px;
            }
            @media screen and (max-width: 767px) {
              background-size: contain;
            }
          }
        }
      }
      .friends_page_banner_bg_img {
        background-image: url(/images/friends_banner_bg.png);
        background-position: center;
        background-repeat: no-repeat;
        background-size: contain;
        width: 1000px;
        height: 1000px;
        position: absolute;
        top: -252px;
        z-index: 0;
        margin: 0 auto;
        left: -600px;
        right: 0;
        filter: opacity(0.4);
        @media screen and (max-width: 767px) {
          left: -80px;
          width: 600px;
          filter: none;
        }
      }
      .right_stars_bg {
        background-image: url(/images/friends_banner_img_stars.png);
        background-position: center;
        background-repeat: no-repeat;
        width: 600px;
        height: 100%;
        position: absolute;
        top: 0px;
        left: 0;
        width: 600px;
        margin: 0 auto;
        mix-blend-mode: screen;
        z-index: 2;
        background-size: auto 307px;
        right: -573px;
        background-position: center calc(100% - 10px);
        @media screen and (max-width: 767px) {
          right: 0;
          background-size: contain;
          width: 100%;
          height: 340px;
          bottom: 0;
          top: auto;
        }
      }
    }

    .boxshadow_container {
      box-shadow: 0px 0px 20px rgba(15, 220, 221, 0.2) !important;
      background: rgba(0, 0, 0, 0) !important;
      padding-bottom: 20px !important;
    }

    > .friends_page_content {
      position: relative;
      z-index: 1;
      background-image: url(/ssm/public/images/bg.jpg);
      background-size: cover;
      background-repeat: no-repeat;
      background-position: top;
      .card_content {
        padding-top: 50px;
        padding-bottom: 35px;
        .title_container {
          color: #3467F0;
          font-size: 30px;
          padding-bottom: 30px;
          text-align: center;
        }

        .invite_link_container {
          width: 1000px;
          max-width: 95%;
          display: flex;
          justify-content: center;
          padding-top: 35px;
          .invite_link_block {
            width: 50%;
            padding: 15px 15px;
            display: flex;
            flex-direction: column;
            align-content: center;
            align-items: center;
            .title_ {
              font-size: 20px;
              font-weight: 600;
              color: #5B5B5B;
              .small_ {
                font-size: 14px;
                font-weight: normal;
              }
            }
            .share_inout_container {
              margin: 10px 0px;
              position: relative;
              input, .share_email_input {
                height: auto;
                /* left: 61px; */
                top: 110px;
                background: #FFFFFF;
                box-shadow: inset 0px 4px 10px rgba(177, 181, 223, 0.4);
                border-radius: 50px;
                width: 350px;
                padding: 10px;
                font-size: 17px;
                padding-left: 20px;
                padding-right: 100px;
              }

              

              .share_input_btn {
                position: absolute;
                z-index: 2;
                cursor: pointer;
                background: #00D8CD;
                color: #fff;
                right: 0;
                top: 0;
                border-radius: 0px 50px 50px 0px;
                font-size: 17px;
                padding: 10px;
                padding-left: 21px;
                padding-right: 25px;
                min-height: 45.5px;
                box-shadow: none;
                max-height: 45.5px;
                width: 87px;
              }
            }
            .social_share_list_container {
              display: flex;
              margin-top: 15px;
              align-items: center;
              .social_share_title {
                color: #15D6DF;
                font-size: 20px;
                margin-right: 30px;
              }
              .social_share_container {
                box-shadow: none;
                width: 35px;
                height: 35px;
                margin: 0;
                max-width: 35px;
                min-width: 35px;
                margin-right: 5px;
                margin-left: 5px;
                background: rgba(0,0,0,0);
                i {
                  background-image: url(/images/LINE_logo.png);
                  background-size: contain;
                  background-repeat: no-repeat;
                  background-position: center;
                  width: 35px;
                  height: 35px;
                }
                &:nth-child(3) i {
                  background-image: url(/images/Facebook_logo.png);
                }
                &:nth-child(4) i {
                  background-image: url(/images/Twitter-X-Icon.png);
                }
                &:nth-child(5) i {
                  background-image: url(/images/WhatsApp_logo.png);
                }
              }
            }
            .comment_container {
              font-size: 14px;
              color: #638FF9;
              margin-left: -96px;
              margin-top: -6px;
            }
            
          }

          
        }

        @media screen and (max-width: 767px) {
          .invite_link_container {
            flex-direction: column;
            .invite_link_block {
              width: 100%;
              .share_inout_container input {
                width: 300px;
              }
              .social_share_list_container {
                margin-bottom: 30px;
              }
              .comment_container {
                margin-left: 0;
              }
            }
          }
        }

        &.card_2 {
          background-image: url(/images/bg.jpg);
          background-position: center center;
          background-repeat: no-repeat;
          background-size: cover;
          .mission_list_container {
            display: flex;
            margin: 0 auto;
            width: 1000px;
            max-width: 95%;
            .mission_container {
              position: relative;
              padding: 20px;
              padding-top: 40px;
              margin-right: 50px;
              .step_count_bg {
                position: absolute;
                width: 100%;
                height: 100%;
                background-image: url(/images/friends_step_1.svg);
                background-repeat: no-repeat;
                background-size: contain;
                background-position: center;
                height: 160px;
                width: 160px;
                margin: 0 auto;
                left: 0;
                right: 0;
                filter: opacity(0.3);
              }
              .mission_content {
                position: relative;
                z-index: 2;
                color: #5B5B5B;
                font-size: 18px;
                text-align: center;
                .title_ {
                  font-size: 23px;
                  font-weight: bold;
                  letter-spacing: 2px;
                  text-indent: 2px;
                }
                .content_ {
                  padding-top: 30px;
                  padding-bottom: 30px;
                  height: 141px;
                }
                .bottom_icon {
                  padding-bottom: 10px;
                  text-align: center;
                  
                  i {
                    width: 50px;
                    height: 50px;
                    display: inline-block;
                    background-image: url(/images/friends_step_1.png);
                    background-size: contain;
                    background-position: center;
                    background-repeat: no-repeat;
                  }
                }
              }

              &:nth-child(2) {
                .step_count_bg {
                  background-image: url(/images/friends_step_2.svg);
                }
                .bottom_icon i {
                  background-image: url(/images/friends_step_2.png);
                }
              }

              &:nth-child(3) {
                margin-right: 0px;
                .step_count_bg {
                  background-image: url(/images/friends_step_3.svg);
                }
                .bottom_icon i {
                  background-image: url(/images/friends_step_3.png);
                }
              }

            }
          }
          @media screen and (max-width: 767px) {
            .mission_list_container {
              flex-direction: column;
              .mission_container {
                margin-right: 0;
                margin-left: 2.5%;
              }
            }
          }
        }

        &.card_3 {
          @media screen and (max-width: 767px) {
            .title_container {
              display: none;
            }
          }
          .progress_bar_container {
            width: 1000px;
            max-width: 95%;
            display: flex;
            font-size: 17px;
            position: relative;
            justify-content: center;
            color: #7F7F7F;
            padding-left: 150px;
            padding-bottom: 50px !important;
            .bar_container {
              position: absolute;
              margin: 0 auto;
              width: 520px;
              height: 9px;
              background: #ededed;
              left: 0;
              right: -130px;
              top: 93px;
              .bar_proccess {
                width: 0px;
                max-width: 520px;
                height: 100%;
                background: #c6afff;
              }
            }
            .invite_text_float {
              position: absolute;
              margin: 0 auto;
              left: -671px;
              right: 0;
              width: 100px;
              text-align: center;
            }
            .earn_coin_text_float {
              position: absolute;
              margin: 0 auto;
              left: -671px;
              right: 0;
              width: 100px;
              text-align: center;
              top: 127px;
            }
            .step {
              width: 130px;
              position: relative;
              display: flex;
              flex-direction: column;
              align-content: center;
              align-items: center;
              .circle {
                background: #638FF9;
                position: absolute;
                width: 15px;
                height: 15px;
                border-radius: 15px;
                margin: 0 auto;
                left: 0;
                right: 0;
                top: 40px;
              }
              .coin_ {
                margin-top: 49px;
                background: #E9E0FF;
                padding: 3px;
                width: 80px;
                text-align: center;
                border-radius: 10px;
                position: relative;
                &::before {
                  content: "";
                  display: block;
                  width: 0;
                  height: 0;
                  border-bottom: 10px solid #E9E0FF;
                  border-right: 5px solid transparent;
                  border-left: 5px solid transparent;
                  position: absolute;
                  top: -8px;
                  margin: 0 auto;
                  left: 0;
                  right: 0;
                }
                .crown_ {
                  background-image: url(/images/best.png);
                  width: 20px;
                  height: 20px;
                  display: block;
                  background-size: contain;
                  background-position: center;
                  background-repeat: no-repeat;
                  position: absolute;
                  top: -13px;
                  filter: brightness(0) saturate(100%) invert(86%) sepia(21%) saturate(6627%) hue-rotate(334deg) brightness(106%) contrast(87%);
                  right: 0;
                }
              }
            }
            @media screen and (max-width: 767px) {
              display: none;
            }
          }
          .stat_card_container {
            width: 1000px;
            max-width: 95%;
            margin: 0 auto;
            display: flex;
            .stat_card {
              margin-right: 80px;
              /* padding-bottom: 30px !important; */
              padding-bottom: 50px !important;
              display: flex;
              flex-direction: column;
              align-items: center;
              .stat_number {
                color: #00D8CD;
                font-size: 80px;
                line-height: 1em;
              }
              .stat_label {
                display: flex;
                align-items: center;
                font-size: 19px;
                padding-top: 10px;
                height: 45px;
                i.dollar_coin {
                  background-image: url(/images/dollar_coin.svg);
                  background-position: center;
                  background-repeat: no-repeat;
                  background-size: contain;
                  width: 35px;
                  height: 35px;
                  display: inline-block;
                }
              }
              .btn_container_ {
                a.redeem_btn {
                  width: auto;
                  height: auto;
                  padding: 3px 50px;
                  padding-bottom: 6px;
                  margin-top: 6px;
                  margin-bottom: -13px;
                }
              }
              &:last-child {
                margin-right: 0px;
              }
            }
          }
          @media screen and ( max-width: 767px ) {
            .stat_card_container {
              flex-direction: column;
              .stat_card {
                margin-right: 0;
                margin-left: 2.5%;
              }
            }
          }
        }

        &.card_4 {
          position: relative;
          z-index: 2;
          background-image: url(/images/bg.jpg);
          background-position: center center;
          background-repeat: no-repeat;
          background-size: cover;
          .faq_container {
            width: 1000px;
            max-width: 95%;
            background: rgba(255, 255, 255, 0.5) !important;
            padding-bottom: 50px !important;
            .v-expansion-panel {
              background: none;
              color: #5b5b5b;
            }
            h2 {
              z-index: 52;
              font-family: "Noto Sans TC", sans-serif;
              padding: 50px 91px 50px 91px;
              margin-bottom: 72px;
              line-height: 1.4;
              font-size: 48px;
              letter-spacing: 3px;
              text-indent: 3px;
              width: 50%;
              color: #fff;
              font-weight: bold;
              border-radius: 999px 0 0 999px;
              background: linear-gradient(
                to right,
                /* 調整停止點位置 */ rgba(57, 90, 244, 0.4) 0%,
                /* 調整停止點位置 */ rgba(57, 90, 244, 0.4) 20%,
                rgba(0, 216, 205, 0.4) 50%,
                rgba(0, 216, 205, 0.4) 100%
              );
            }

            &-panels {
              position: relative;
              z-index: 53;
              width: 55%;
              background-color: transparent;
              :deep(.v-expansion-panel) {
                border-radius: 0px;
                background-color: transparent;
                border-bottom: 1px solid #000;

                &:last-child {
                  border-bottom: 0px;
                }
              }
            }

            &-circle {
              position: absolute;
              z-index: 52;
              top: -230px;
              right: -5%;
              width: 1200px;
            }

            h4 {
              line-height: 1.5;
              font-weight: bold;
              font-size: 18px;
              letter-spacing: 1px;
            }

            @media screen and (max-width: 1366px) and (min-width: 1000px) {
              
              h2 {
                padding: 50px 91px 50px 91px;
                margin-bottom: 72px;
                line-height: 1.4;
                font-size: 40px;
                width: 50%;
              }

              &-panels {
                width: 55%;
              }

              &-circle {
                top: -130px;
                right: -8%;
                width: 800px;
              }

              h4 {
                font-size: 20px;
                line-height: 1.5;
              }
            }
            @media screen and (max-width: 999px) and (min-width: 768px) {
              h2 {
                padding: 50px 91px 50px 91px;
                margin-bottom: 72px;
                line-height: 1.4;
                font-size: 36px;
                width: 50%;
              }

              &-panels {
                width: 80%;
              }

              &-circle {
                top: -130px;
                right: -35%;
                width: 770px;
              }

              h4 {
                font-size: 18px;
                line-height: 1.5;
              }
            }
            @media screen and (max-width: 767px) {
              padding-top: 0px;
              h2 {
                padding: 32px 60px 32px 60px;
                margin-bottom: 72px;
                line-height: 1.4;
                font-size: 20px;
                width: 50%;
              }

              &-panels {
                width: 80%;
              }

              &-circle {
                top: -73px;
                right: -68%;
                width: 510px;
              }

              h4 {
                font-size: 16px;
                line-height: 1.5;
              }

              button {
                padding: 16px 5px;
              }

              ._content  {
                padding: 0;
              }

            }
          }
        }

        &.card_5 {
          position: relative;
          z-index: 1;
          overflow: hidden;
          .card_5_bg {
            background-image: url(/images/friends_btm_bg_icon.svg);
            background-size: contain;
            background-repeat: no-repeat;
            background-position: center;
            position: absolute;
            width: 1700px;
            height: 1200px;
            z-index: 0;
            background-repeat: no-repeat;
            top: -214px;
            left: -400px;
            right: 0;
            margin: 0 auto;
            filter: opacity(0.6);
          }
          .title_container {
            position: relative;
            z-index: 1;
          }
          .friends_container {
            position: relative;
            z-index: 1;
            width: 1000px;
            max-width: 95%;
            padding: 50px;
            padding-left: 70px;
            padding-top: 60px;
            padding-bottom: 50px !important;
            background: rgba(255,255,255,0.5) !important;
            ul.friends_content {
              padding: 0;
              margin: 0;
              font-size: 15px;
              color: #5b5b5b;
              line-height: 2em;
            }
            @media screen and ( max-width: 767px ) {
              padding: 20px;
              padding-left: 40px;
              padding-bottom: 20px !important;
              ul.friends_content {
                line-height: 1.6em;
              }
            }
          }

        }
      }

      .footer_container {
        background: linear-gradient(360deg, rgba(56, 91, 245, 0.7) 0.18%, rgba(11, 211, 212, 0.7) 99.43%);
        /* transform: rotate(-180deg); */
        padding: 27px;
        width: 100%;
        position: relative;
        .footer_content {
          width: 500px;
          font-size: 14px;
          color: #fff;
          text-align: center;
          margin: 0 auto;
          a {
            color: #fff;
          }
          @media screen and ( max-width: 767px ) {
            width: 95%;
          }
        }
      }

      .friends_page {
        max-width: 1200px;
        width: 95%;
        position: relative;
        margin: 0 auto;
        line-height: 2em;
        .comment_container {
          width: 100%;
          text-align: right;
          font-size: 14px;
          padding-bottom: 20px;
        }
      }
    }

    
  }

  
.friends_page_modal_container {
  .v-overlay__content {
    .v-card {
      position: relative;
      background: #ddd0ff;
      border-radius: 20px;
      .close_btn_container {
        position: absolute;
        right: 5px;
        button {
          background: 0;
          box-shadow: none;
          font-size: 20px;
          width: 20px;
        }
      }
      .dialog_content {
        padding-top: 30px;
        padding-bottom: 30px;
        font-size: 16px;
        font-weight: 500;
        width: 450px;
        max-width: 90%;
        margin: 0 auto;
        text-align: center;
        &.align_left {
          text-align: left;
        }
        ._highlight {
          font-weight: bold;
          color: #3150f3;
        }
        .small_highlight {
          font-size: 15px;
          color: #638ff9;
        }
        .button_container {
          display: block;
          padding-top: 15px;
          text-align: center;
          button {
            height: 40px;
            font-size: 15px;
            border: none;
            box-shadow: none;
            background-color: #00d8cd;
            border: 2px solid #00d8cd;
            border-radius: 20px;
            color: #fff;
            margin-right: 10px;
            &:hover {
              border: 2px solid #638ff9;
              color: #638ff9;
              background: none;
            }
            &:last-child {
              margin-right: 0px;
            }
          }
        }
      }
    }
  }
}

// 方案資訊dialog
.level_intro_modal_container_2 {
  .v-overlay__content {
    .v-card {
      position: relative;
      background: #ddd0ff;
      border-radius: 20px;
      padding: 0 !important;
      .close_btn_container {
        position: absolute;
        right: 5px;
        button {
          color: #000;
          background: 0;
          box-shadow: none;
          font-size: 20px;
          width: 20px;
        }
      }
      .dialog_content {
        padding-top: 30px;
        padding-bottom: 30px;
        font-size: 16px;
        font-weight: 500;
        width: 450px;
        max-width: 90%;
        margin: 0 auto;
        text-align: center;
        &.align_left {
          text-align: left;
        }
        ._highlight {
          font-weight: bold;
          color: #3150f3;
        }
        .small_highlight {
          font-size: 15px;
          color: #638ff9;
        }
        .button_container {
          display: block;
          padding-top: 15px;
          text-align: center;
          button {
            width: auto;
            height: 40px;
            font-size: 15px;
            border: none;
            box-shadow: none;
            background-color: #00d8cd;
            border: 2px solid #00d8cd;
            border-radius: 20px;
            color: #fff;
            margin-right: 10px;
            &:hover {
              border: 2px solid #638ff9;
              color: #638ff9;
              background: none;
            }
            &:last-child {
              margin-right: 0px;
            }
          }
        }
      }
    }
  }
  &.delete_user_dialog_container {
    .v-overlay__content .v-card {
      background: #fff;
      width: 600px;
      .dialog_content {
        padding-top: 50px;
        .button_container {
          padding-top: 30px;
          button {
            width: 150px;
            &:nth-child(2) {
              background-color: #ff6868;
              border: 2px solid #ff6868;
              &:hover {
                background: rgba(0, 0, 0, 0);
                border: 2px solid #ff6868;
                color: #ff6868;
              }
            }
          }
        }
      }
    }
  }
  &.video_view_modal_container {
    .v-overlay__content {
      width: calc(450px / 9 * 16) !important;
      height: 500px;
    }
    .v-overlay__content .v-card {
      background: #fff;
      .dialog_content {
        width: 100%;
        height: 450px;
        iframe {
          width: 100%;
          height: 100%;
        }
      }
    }
  }
}



.fixed-right-btn {
  position: fixed;
  top: 100px;
  right: 0;
  margin-right: 4.5%;
  z-index: 100;
  .v-btn:nth-child(1) {
    background: #a9c6fe;
    box-shadow: 5px 5px 1px #8ab2ff;
    &:hover {
      transform: scale(1.1);
    }
  }
  .v-btn:nth-child(2) {
    background: #00d8cd;
    box-shadow: 5px 5px 1px #1a8d87;
    border: 2px solid #00d8cd;
    &:hover {
      border: 2px solid #1a8d87;
      color: #1a8d87;
      background: #fff;
    }
    font-size: 15px;
  }
  .v-img {
    width: 64px;
    height: 64px;
  }
  .v-btn {
    width: 64px;
    height: 64px;
    min-width: 0;
  }
}

.__lang-en {
  .friends_page_banner_container .friends_page_banner {
    transform: none; /* 取消放大，避免長字撐爆 */
    min-height: 380px; /* 降低最小高度 */

    .left_container {
      .top_title {
        font-size: 18px;
        margin-bottom: 10px;
        white-space: nowrap; /* 英文一行顯示 */
      }
      .middle_title {
        .title_1_text,
        .title_2_text,
        .title_2_text_light_mode_ {
          font-size: 30px;
          line-height: 1.1;
        }
      }
      .bottom_comment {
        font-size: 16px;
        line-height: 1.4;
      }
    }

    .right_container .kerker_img {
      background-size: auto 260px;
    }

    @media screen and (max-width: 767px) {
      min-height: auto;

      .left_container {
        .middle_title {
          .title_1_text,
          .title_2_text,
          .title_2_text_light_mode_ {
            font-size: 26px;
          }
        }
        .bottom_comment {
          font-size: 14px;
        }
      }
    }
  }
}

.lang-en {
  .friends_page_container .friends_page_banner_container .friends_page_banner .left_container .top_title {
    letter-spacing: 0px;
    font-size: 17px;
  }

  .friends_page_container .friends_page_banner_container .friends_page_banner .left_container .middle_title .title_1_text {
    font-size: 30px;
    letter-spacing: 0px;
  }

  .friends_page_container .friends_page_banner_container .friends_page_banner .left_container .middle_title .title_2_text {
    font-size: 30px;
    letter-spacing: 0px;
  }

  .friends_page_container .friends_page_banner_container .friends_page_banner .left_container .middle_title .title_2_text_light_mode_ {
    font-size: 30px;
    letter-spacing: 0px;
  }

  .friends_page_container .friends_page_banner_container .friends_page_banner .left_container .bottom_comment {
    font-size: 16px;
  }

  .friends_page_container > .friends_page_content .card_content.card_3 .stat_card_container .stat_card .stat_label {
    height: 60px;
    line-height: 1.2em;
    text-align: center;
  }
}

</style>
