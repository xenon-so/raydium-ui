<template>
  <div class="sliding_card_warper" v-bind:class="{ active_card: isActive }">
    <div class="sliding_card_switch" v-on:click="isActive = !isActive">
      <img src="../assets/icons/acceleRaytor.svg" width="20" height="20" />
    </div>
    <div class="sliding_card_inner borrow_account_status_wrapper">
      <h2>Margin Account</h2>
      <div class="account-status-list">
        <p>Assets</p>
        <span>${{ assetValue }}</span>
      </div>
      <div class="account-status-list">
        <p>Margin available</p>
        <span>${{ usdcLeftToBorrow }}</span>
      </div>
      <div class="account-status-list">
        <p>Leverage</p>
        <span>{{ accountLeverage }}x</span>
      </div>
      <div class="progress-bar-wrapper">
        <p>Health</p>
        <Progress :percent="healthPercent" />
      </div>
      <div class="account-status-list">
        <p>USDC</p>
        <span>{{ usdcBalance }}</span>
      </div>
      <div class="account-status-list">
        <p>BTC</p>
        <span>{{ btcBalance }}</span>
      </div>
      <div class="account-status-list">
        <p>ETH</p>
        <span>{{ ethBalance }}</span>
      </div>
      <div class="account-status-list">
        <p>WSOL</p>
        <span>{{ wsolBalance }}</span>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { mapState } from 'vuex'
// import { Progress, } from 'ant-design-vue'
// import 'ant-design-vue/dist/antd.css';

import { Progress, Collapse } from 'ant-design-vue'
import { PublicKey } from '@solana/web3.js'
import { ACCOUNT_LAYOUT, MARGIN_DATA, XENON_DATA } from '../utils/investinLayout'
import * as BufferLayout from 'buffer-layout'

Vue.use(Progress)

const programId = new PublicKey('9RZ8Fgh2n5urcfkUw6jDHBn1DnrDaiitc9uuc8UevMeY')
export const MAX_RATE = 4.75 * 10 ** -8
export const MAX_RATE_YEAR = 1.5
export const OPTIMAL_UTIL = 0.7
export const OPTIMAL_RATE = 1.9 * 10 ** -9
export const OPTIMAL_RATE_YEAR = 0.06
const TOKEN_PROGRAM_ID = new PublicKey('TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA')

const TOKENS = {
  WSOL: {
    symbol: 'WSOL',
    name: 'Wrapped Solana',
    mintAddress: 'So11111111111111111111111111111111111111112',
    decimals: 9,
    referrer: 'HTcarLHe7WRxBQCWvhVB8AP56pnEtJUV2jDGvcpY3xo5',
    tags: ['raydium']
  },
  BTC: {
    symbol: 'BTC',
    name: 'Wrapped Bitcoin',
    mintAddress: '3UNBZ6o52WTWwjac2kPUb4FyodhU1vFkRJheu1Sh2TvU',
    decimals: 6,
    referrer: 'GZpS8cY8Nt8HuqxzJh6PXTdSxc38vFUjBmi7eEUkkQtG',
    tags: ['raydium']
  },
  ETH: {
    symbol: 'ETH',
    name: 'Wrapped Ethereum',
    mintAddress: 'Cu84KB3tDL6SbFgToHMLYVDJJXdJjenNzSKikeAvzmkA',
    decimals: 6,
    referrer: 'CXPTcSxxh4AT38gtv3SPbLS7oZVgXzLbMb83o4ziXjjN',
    tags: ['raydium']
  },
  USDC: {
    symbol: 'USDC',
    name: 'USD Coin',
    mintAddress: '8FRFC6MoGGkMFQwngccyu69VnYbzykGeez7ignHVAFSN',
    decimals: 6,
    referrer: 'CXPTcSxxh4AT38gtv3SPbLS7oZVgXzLbMb83o4ziXjjN',
    tags: ['raydium']
  }
}

export default Vue.extend({
  components: {
    Progress
  },
  data() {
    return {
      isActive: true,
      assets: 0,
      percent: 0,
      usdcLeftToBorrow: 0,
      leverage: 0,
      text: `A dog is a type of domesticated animal.Known for its loyalty and faithfulness,it can be found as a welcome guest in many households across the world.`,
      activeKey: ['1'],
      xenonProgramData: {
        total_borrows: 0,
        total_deposits: 0,
        last_updated: '0',
        borrow_index: 0
      },
      accountMarginData: {
        assets: 0,
        liabs: 0
      },
      liabilities: 0,
      accountLeverage: 0,
      borrowAPR: 0,
      assetValue: 0,
      healthPercent: 0,
      usdcBalance: 0,
      btcBalance: 0,
      ethBalance: 0,
      wsolBalance: 0
    }
  },
  watch: {
    activeKey(key) {
      console.log(key)
    },
    'wallet.tokenAccounts': {
      handler(newTokenAccounts: any) {
        this.checkIfWalletExists()
      },
      deep: true
    },
    usdcBalance: function (val) {
      this.checkIfWalletExists()
    },
    btcBalance: function (val) {
      this.checkIfWalletExists()
    },
    ethBalance: function (val) {
      this.checkIfWalletExists()
    },
    wsolBalance: function (val) {
      this.checkIfWalletExists()
    }
  },
  computed: {
    ...mapState(['wallet'])
  },
  mounted() {
    this.checkIfWalletExists()
  },
  methods: {
    async getPricesFromCoingecko() {
      const res = await fetch(
        'https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&ids=bitcoin%2Csolana%2Cethereum%2Cserum&order=market_cap_desc&per_page=100&page=1&sparkline=false'
      )
      const data = await res.json()
      return data
    },
    async getTotalAssets(usdcBalance: number, btcBalance: number, ethBalance: number, wsolBalance: number) {
      const prices = await this.getPricesFromCoingecko()
      const btcTotal = btcBalance * prices.find((t: any) => t.id === 'bitcoin').current_price
      const ethTotal = ethBalance * prices.find((t: any) => t.id === 'ethereum').current_price
      const solTotal = wsolBalance * prices.find((t: any) => t.id === 'solana').current_price
      return parseFloat(`${usdcBalance}`) + btcTotal + ethTotal + solTotal
    },
    getOwnedAccountsFilters(publicKey: PublicKey) {
      return [
        {
          memcmp: {
            offset: ACCOUNT_LAYOUT.offsetOf('owner'),
            bytes: publicKey.toBase58()
          }
        },
        {
          dataSize: ACCOUNT_LAYOUT.span
        }
      ]
    },
    mapTokens(resps: any) {
      const tokens = []
      for (const res of resps) {
        tokens.push({
          pubkey: res.pubkey,
          data: ACCOUNT_LAYOUT.decode(res.account.data)
        })
      }
      return tokens
    },
    roundDownTo4Decimals(value: any) {
      if (value) {
        return parseFloat(`${Math.floor(value * 10000) / 10000}`).toFixed(4)
      } else {
        return 0
      }
    },
    async checkIfWalletExists() {
      if (this.$wallet) {
        const xenonPDA = await PublicKey.findProgramAddress([Buffer.from('xenon_v1')], programId)
        let xenonInfo = await this.$web3.getAccountInfo(xenonPDA[0])

        if (xenonInfo) {
          let xenonData = XENON_DATA.decode(xenonInfo.data)
          this.xenonProgramData = xenonData

          const xenonProgramDataSubscription = this.$web3.onAccountChange(
            xenonPDA[0],
            (x, y) => {
              const data = XENON_DATA.decode(x.data)
              this.xenonProgramData = data
            },
            'recent'
          )
        }
        // const marginPDA = await PublicKey.findProgramAddress([this.$wallet?.walletPublicKey.toBuffer()], programId)
        const marginPDA = new PublicKey('78FDFHPP4PTzCaLfXUKt9vZcPELLvnnS3wnQeGNwRAsm');
        let marginInfo = await this.$web3.getAccountInfo(marginPDA)
        if (marginInfo) {
          let marginData = MARGIN_DATA.decode(marginInfo.data)
          this.accountMarginData = marginData

          const marginAccountSubscription = this.$web3.onAccountChange(
            marginPDA,
            (x, y) => {
              const data = MARGIN_DATA.decode(x.data)
              this.accountMarginData = data
            },
            'recent'
          )
        }

        let borrowRate = 0
        if (this.xenonProgramData) {
          const totalBorrows = this.xenonProgramData.total_borrows
          const totalDeposits = this.xenonProgramData.total_deposits

          if (totalDeposits === 0 && totalBorrows === 0) {
            borrowRate = 0
          }
          if (totalDeposits <= totalBorrows) {
            borrowRate = MAX_RATE
          }

          const utilization = totalBorrows / totalDeposits
          if (utilization > OPTIMAL_UTIL) {
            const extraUtil = utilization - OPTIMAL_UTIL
            const slope = (MAX_RATE - OPTIMAL_RATE) / (1 - OPTIMAL_UTIL)
            borrowRate = OPTIMAL_RATE + slope * extraUtil
          } else {
            const slope = OPTIMAL_RATE / OPTIMAL_UTIL
            borrowRate = slope * utilization
          }
        } else {
          borrowRate = 0
        }

        const totalAssets = await this.getTotalAssets(
          this.usdcBalance,
          this.btcBalance,
          this.ethBalance,
          this.wsolBalance
        )

        const assets = totalAssets
        const currentDate = Math.floor(Date.now() / 1000)
        const lastUpdatedTime = (this.xenonProgramData.last_updated as any).toNumber()
        const borrowIndex =
          this.xenonProgramData.borrow_index +
          this.xenonProgramData.borrow_index * borrowRate * (currentDate - lastUpdatedTime)

        const liabs = (this.accountMarginData.liabs / 10 ** 6) * borrowIndex

        this.liabilities = liabs

        this.usdcLeftToBorrow = Number((assets - 2 * liabs < 0 ? 0 : assets - 2 * liabs).toFixed(2))

        const coll_ratio = assets / liabs
        this.accountLeverage = Number((1 / (coll_ratio - 1) + 1).toFixed(2))
        this.borrowAPR = Number((borrowRate * 3.154e7 * 100).toFixed(2))
        this.assetValue = totalAssets

        this.healthPercent = Number(((assets / (this.accountMarginData.liabs / 10 ** 6) - 1.2) * 100).toFixed(0))

        let filters = this.getOwnedAccountsFilters(marginPDA)
        let resp = await this.$web3.getProgramAccounts(TOKEN_PROGRAM_ID, {
          filters
        })

        const mappedTokens = this.mapTokens(resp)
        try {
          const token = mappedTokens.find((t) => t.data.mint.toBase58() === TOKENS.USDC.mintAddress)
          const accounts = token?.pubkey
          const walletBalance = await this.$web3.getTokenAccountBalance(accounts, 'max')
          this.usdcBalance = Number(walletBalance.value.uiAmountString) ?? 0
          const USDCAccountSubscription = this.$web3.onAccountChange(
            accounts,
            (x, y) => {
              const USDCAccState = ACCOUNT_LAYOUT.decode(x.data)
              const bal = this.roundDownTo4Decimals(USDCAccState.amount.toNumber() / 10 ** TOKENS.USDC.decimals)
              this.usdcBalance = Number(bal)
              // console.log("state changed ::", USDCAccState, bal)
            },
            'recent'
          )
        } catch (error) {
          console.error('error reading usdc account :>> ', error)
        }

        try {
          const token = mappedTokens.find((t) => t.data.mint.toBase58() === TOKENS.WSOL.mintAddress)
          const accounts = token?.pubkey
          const walletBalance = await this.$web3.getTokenAccountBalance(accounts, 'max')
          this.wsolBalance = Number(walletBalance.value.uiAmountString)

          const USDCAccountSubscription = this.$web3.onAccountChange(
            accounts,
            (x, y) => {
              const USDCAccState = ACCOUNT_LAYOUT.decode(x.data)
              const bal = this.roundDownTo4Decimals(USDCAccState.amount.toNumber() / 10 ** TOKENS.WSOL.decimals)
              this.wsolBalance = Number(bal)
              // console.log("state changed ::", USDCAccState, bal)
            },
            'recent'
          )
        } catch (error) {
          console.error('error reading WSOL account :>> ', error)
        }

        try {
          const token = mappedTokens.find((t) => t.data.mint.toBase58() === TOKENS.BTC.mintAddress)
          const accounts = token?.pubkey
          const walletBalance = await this.$web3.getTokenAccountBalance(accounts, 'max')
          this.btcBalance = Number(walletBalance.value.uiAmountString)

          const USDCAccountSubscription = this.$web3.onAccountChange(
            accounts,
            (x, y) => {
              const USDCAccState = ACCOUNT_LAYOUT.decode(x.data)
              const bal = this.roundDownTo4Decimals(USDCAccState.amount.toNumber() / 10 ** TOKENS.BTC.decimals)
              this.btcBalance = Number(bal)
              // console.log("state changed ::", USDCAccState, bal)
            },
            'recent'
          )
        } catch (error) {
          console.error('error reading BTC account :>> ', error)
        }

        try {
          const token = mappedTokens.find((t) => t.data.mint.toBase58() === TOKENS.ETH.mintAddress)
          const accounts = token?.pubkey
          const walletBalance = await this.$web3.getTokenAccountBalance(accounts, 'max')
          this.ethBalance = Number(walletBalance.value.uiAmountString)

          const USDCAccountSubscription = this.$web3.onAccountChange(
            accounts,
            (x, y) => {
              const USDCAccState = ACCOUNT_LAYOUT.decode(x.data)
              const bal = this.roundDownTo4Decimals(USDCAccState.amount.toNumber() / 10 ** TOKENS.ETH.decimals)
              this.ethBalance = Number(bal)
              // console.log("state changed ::", USDCAccState, bal)
            },
            'recent'
          )
        } catch (error) {
          console.error('error reading ETH account :>> ', error)
        }
      }
    }
  }
})
</script>

<style lang="less" scoped>
@import '../styles/variables';

.select-token {
  display: grid;
  grid-auto-rows: auto;
  row-gap: 14px;

  .token-list {
    max-height: 60vh;
    padding-right: 10px;
    overflow: auto;
    direction: ltr;
    will-change: transform;

    .token-info {
      width: 100%;
      justify-content: space-between;
      padding: 4px 0;
      cursor: pointer;
      opacity: 1;
      height: 70px;
      &:hover,
      &:active {
        background: rgb(107, 128, 185);
      }
    }

    .token-info[disabled] {
      cursor: not-allowed;
      // pointer-events: none;
      opacity: 0.5;
    }
  }
}

.sliding_card_warper {
  width: 316px;

  /* height: 100%; */
  position: fixed;
  display: flex;
  align-items: center;
  top: 0;
  left: 0;
  padding: 15% 0;
  z-index: 99;
  transition: all ease-in-out 0.4s;
}

.sliding_card_warper.active_card {
  left: -316px;
}

.sliding_card_switch {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  width: 46px;
  height: 78.43px;
  left: 316px;
  top: calc(50% - 22px);
  background: radial-gradient(
    166.67% 155% at -84.31% -26.5%,
    rgba(255, 255, 255, 0.06) 0%,
    rgba(255, 255, 255, 0) 100%
  );
  border: 1px solid rgba(204, 204, 204, 0.24);
  border-left: none;
  box-shadow: inset -5px -5px 250px rgba(255, 255, 255, 0.02);
  backdrop-filter: blur(20px);

  /* Note: backdrop-filter has minimal browser support */
  border-radius: 0 14px 14px 0;
  z-index: 9;
}

.sliding_card_inner {
  width: 316px;
  max-height: 660px;
  height: 100%;
  background: radial-gradient(100% 100% at 0% 0%, rgba(255, 255, 255, 0.06) 0%, rgba(255, 255, 255, 0) 100%);
  border: 1px solid rgba(204, 204, 204, 0.24);
  box-shadow: inset -5px -5px 250px rgba(255, 255, 255, 0.02);
  backdrop-filter: blur(20px);
  border-radius: 0 21px 21px 0;
  padding: 25px 28px;
  overflow-y: auto;
}

.sliding_card_inner h2 {
  color: #fff;
  font-weight: 600;
  font-size: 20px;
  padding-bottom: 13px;
  margin: 0 -28px;
  text-align: center;
  border-bottom: 1px solid rgba(204, 204, 204, 0.24);
}

.sliding_card_inner .account-status-list,
.sliding_card_inner .sliding-card-panel-head {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  color: #fff;
  border-bottom: 1px solid rgba(117, 117, 117, 0.15);
  padding: 10px 0;
}

.sliding_card_inner .account-status-list p,
.sliding_card_inner .sliding-card-panel-head p {
  font-family: 'Maven Pro', sans-serif !important;
  font-style: normal;
  font-weight: normal;
  font-size: 16px;
  margin-bottom: 0;
}

.sliding_card_inner .account-status-list span,
.sliding_card_inner .sliding-card-panel-head span {
  font-family: 'Exo 2', sans-serif;
  font-style: normal;
  font-weight: 600;
  font-size: 16px;
}

.sliding_card_inner .sliding-card-panel-head {
  padding: 0;
  padding-right: 28px;
  border-bottom: none;
}

.sliding_card_inner .account-status-list:first-child,
.sliding_card_inner .sliding-card-panel-head:first-child {
  padding-top: 0;
}

.sliding_card_inner .account-status-list.leverage,
.sliding_card_inner .sliding-card-panel-head.leverage {
  border-bottom: none;
}

.sliding_card_inner .progress-bar-wrapper {
  margin-bottom: 30px;
  padding-top: 10px;
}

.sliding_card_inner .progress-bar-wrapper p {
  margin-bottom: 8px;
  font-style: normal;
  font-weight: normal;
  font-size: 14px;
  color: #fff;
}

.sliding_card_inner .ant-progress-inner {
  background-color: #25303b;
  height: 15px;
}

.ant-progress-bg .ant-progress-bg {
  background-color: #369126 !important;
  height: 15px !important;
}

.sliding_card_inner .ant-progress-inner .ant-progress-bg {
  background-color: #369126 !important;
  height: 15px !important;
}

.sliding_card_inner .sliding-card-btn {
  background: linear-gradient(90.33deg, #52c33d -97.42%, #2b7d1c 146.99%);
  border-radius: 10px;
  width: 100%;
  border: 0;
  color: white;
  font-weight: 700;
  font-size: 18px;
  height: 58px;
  cursor: pointer;
}

.ant-collapse-arrow {
  display: none !important;
}

.sliding_card_inner .ant-collapse > .ant-collapse-item > .ant-collapse-header .ant-collapse-arrow {
  color: #40a42e;
  position: absolute;
  right: 2px;
  top: 12px;
  font-size: 15px;
}

.sliding_card_inner .sliding-card-panel-head .state-img {
  position: absolute;
  left: 0;
  top: 0;
  width: 24px;
  height: 24px;
  border-radius: 40px;
}

.sliding_card_inner .sliding-card-panel-head h4 {
  position: relative;
  padding-left: 30px;
  color: #fff;
  margin-bottom: 0;
}

.sliding_card_inner .sliding-card-panel-head span {
  font-size: 14px;
  font-weight: 400;
}

.sliding_card_inner .ant-collapse > .ant-collapse-item > .ant-collapse-header {
  padding: 7px 8px;
}

.sliding_card_inner .ant-collapse-item {
  background: radial-gradient(100% 100% at 0% 0%, rgba(255, 255, 255, 0.06) 0%, rgba(255, 255, 255, 0) 100%);
  border: 1px solid rgba(204, 204, 204, 0.24) !important;
  box-shadow: inset -5px -5px 250px rgba(255, 255, 255, 0.02);
  backdrop-filter: blur(20px);
  border-radius: 19px !important;
  margin-bottom: 18px;
}

.sliding_card_inner .sliding-card-table {
  width: 100%;
  border-top: 1px solid rgba(117, 117, 117, 0.15);
}

.sliding_card_inner .sliding-card-table tr th {
  color: #fff;
  padding-top: 8px;
}

.sliding_card_inner .sliding-card-table tr td {
  color: #fff;
  text-align: center;
}
</style>
