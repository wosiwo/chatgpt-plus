<template>
  <div class="app-background" theme="dark">
    <div class="member">
      <div class="title" style="padding-top: 50px">
        个人中心
      </div>
      <div class="inner" :style="{height: listBoxHeight + 'px'}">

        <el-row>
          <div class="user-profile">
            <user-profile/>
            <el-row class="user-opt" :gutter="20">
              <el-col :span="12">
                <el-button type="primary" @click="showPasswordDialog = true">修改密码</el-button>
              </el-col>
<!--              <el-col :span="12">-->
<!--                <el-button type="primary" v-if="enableReward" @click="showRewardDialog = true">加入众筹</el-button>-->
<!--              </el-col>-->
<!--              <el-col :span="12">-->
<!--                <el-button type="primary" v-if="enableReward" @click="showRewardVerifyDialog = true">众筹核销-->
<!--                </el-button>-->
<!--              </el-col>-->
              <el-col :span="24" style="padding-bottom: 50px;">
                <el-button type="danger" round @click="logout">退出登录</el-button>
              </el-col>
            </el-row>
          </div>
        </el-row>
      </div>
      <login-dialog :show="showLoginDialog" @hide="showLoginDialog = false"/>
      <password-dialog v-if="isLogin" :show="showPasswordDialog" width="100%" @hide="showPasswordDialog = false"
                       @logout="logout"/>
      <reward-verify v-if="isLogin" :show="showRewardVerifyDialog" @hide="showRewardVerifyDialog = false"/>



    </div>
  </div>
</template>

<script setup>
import {onMounted, ref} from "vue"
import {ElMessage} from "element-plus";
import {httpGet, httpPost} from "@/utils/http";
import ItemList from "@/components/ItemList.vue";
import {InfoFilled, SuccessFilled} from "@element-plus/icons-vue";
import LoginDialog from "@/components/LoginDialog.vue";
import {checkSession} from "@/action/session";
import UserProfile from "@/components/UserProfile.vue";
import PasswordDialog from "@/components/PasswordDialog.vue";
import RewardVerify from "@/components/RewardVerify.vue";
import {useRouter} from "vue-router";
import {removeUserToken} from "@/store/session";
import CountDown from "@/components/CountDown.vue";

const listBoxHeight = window.innerHeight - 97
const list = ref([])
const showLoginDialog = ref(false)
const showPayDialog = ref(false)
const vipImg = ref("/images/vip.png")
const enableReward = ref(false) // 是否启用众筹功能
const rewardImg = ref('/images/reward.png')
const qrcode = ref("")
const showPasswordDialog = ref(false);
const showRewardDialog = ref(false);
const showRewardVerifyDialog = ref(false);
const text = ref("")
const user = ref(null)
const isLogin = ref(false)
const router = useRouter()
const curPayProduct = ref(null)
const activeOrderNo = ref("")
const countDownRef = ref(null)
const orderTimeout = ref(1800)
const loading = ref(true)
const orderPayInfoText = ref("")

const payWays = ref({})
const payName = ref("支付宝")
const curPay = ref("alipay") // 当前支付方式


onMounted(() => {
  checkSession().then(_user => {
    user.value = _user
    isLogin.value = true
    httpGet("/api/product/list").then((res) => {
      list.value = res.data
    }).catch(e => {
      ElMessage.error("获取产品套餐失败：" + e.message)
    })
  }).catch(() => {
    router.push("/login")
  })

  httpGet("/api/admin/config/get?key=system").then(res => {
    rewardImg.value = res.data['reward_img']
    enableReward.value = res.data['enabled_reward']
    orderPayInfoText.value = res.data['order_pay_info_text']
    if (res.data['order_pay_timeout'] > 0) {
      orderTimeout.value = res.data['order_pay_timeout']
    }
  }).catch(e => {
    ElMessage.error("获取系统配置失败：" + e.message)
  })

  httpGet("/api/payment/payWays").then(res => {
    payWays.value = res.data
  }).catch(e => {
    ElMessage.error("获取支付方式失败：" + e.message)
  })
})

// refresh payment qrcode
const refreshPayCode = () => {
  if (curPay.value === 'alipay') {
    alipay()
  } else if (curPay.value === 'hupi') {
    huPiPay()
  }
}

const genPayQrcode = () => {
  loading.value = true
  text.value = ""
  httpPost("/api/payment/qrcode", {
    pay_way: curPay.value,
    product_id: curPayProduct.value.id,
    user_id: user.value.id
  }).then(res => {
    showPayDialog.value = true
    qrcode.value = res.data['image']
    activeOrderNo.value = res.data['order_no']
    queryOrder(activeOrderNo.value)
    loading.value = false
    // 重置计数器
    if (countDownRef.value) {
      countDownRef.value.resetTimer()
    }
  }).catch(e => {
    ElMessage.error("生成支付订单失败：" + e.message)
  })
}

const alipay = (row) => {
  payName.value = "支付宝"
  curPay.value = "alipay"
  if (!user.value.id) {
    showLoginDialog.value = true
    return
  }
  if (row) {
    curPayProduct.value = row
  }
  genPayQrcode()
}

// 虎皮椒支付
const huPiPay = (row) => {
  payName.value = payWays.value["hupi"]["name"] === "wechat" ? '微信' : '支付宝'
  curPay.value = "hupi"
  if (!user.value.id) {
    showLoginDialog.value = true
    return
  }
  if (row) {
    curPayProduct.value = row
  }
  genPayQrcode()

}

const queryOrder = (orderNo) => {
  httpPost("/api/payment/query", {order_no: orderNo}).then(res => {
    if (res.data.status === 1) {
      text.value = "扫码成功，请在手机上进行支付！"
      queryOrder(orderNo)
    } else if (res.data.status === 2) {
      text.value = "支付成功，正在刷新页面"
      setTimeout(() => location.reload(), 500)
    } else {
      // 如果当前订单没有过期，继续等待订单的下一个状态
      if (activeOrderNo.value === orderNo) {
        queryOrder(orderNo)
      }
    }
  }).catch(e => {
    ElMessage.error("查询支付状态失败：" + e.message)
  })
}

const logout = function () {
  httpGet('/api/user/logout').then(() => {
    removeUserToken();
    router.push('/login');
  }).catch(() => {
    ElMessage.error('注销失败！');
  })
}

</script>

<style lang="stylus">
@import "@/assets/css/mobile/profile.styl"
</style>
