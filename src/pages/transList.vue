<template>
    <div class="trans-list-wrapper">
        <sec-title class="title" :isShowHelp="false"></sec-title>
        <div class="trans-list-content">
            <tabel-list class="big-trans" :headList="[{
                class: 'tType',
                text: $t('transList.tType.title'),
                cell: 'type'
            },{
                class: 'status',
                text: $t('transList.status.title'),
                cell: 'status'
            },{
                class: 'time',
                text: $t('transList.timestamp'),
                cell: 'date'
            },{
                class: 'address',
                text: $t('transList.tAddress'),
                cell: 'transAddr'
            },{
                class: 'sum',
                text: $t('transList.sum'),
                cell: 'amount'
            },{
                class: 'token',
                text: 'Token',
                cell: 'tokenSymbol'
            }]" :contentList="transList" :clickRow="goDetail">
                <pagination class="__tb_pagination" :currentPage="currentPage + 1" 
                            :totalPage="totalPage" :toPage="toPage"></pagination>
            </tabel-list>

            <tabel-list class="small-trans" :headList="[{
                class: 'tType',
                text: $t('transList.tType.symbol'),
                cell: 'smallType'
            },{
                class: 'address',
                text: $t('transList.tAddress'),
                cell: 'smallTransAddr'
            },{
                class: 'sum',
                text: $t('transList.sum'),
                cell: 'smallAmount'
            }]" :contentList="transList" :clickRow="goDetail">
                <pagination class="__tb_pagination" :currentPage="currentPage + 1" 
                            :totalPage="totalPage" :toPage="toPage"></pagination>
            </tabel-list>
        </div>
    </div>
</template>

<script>
import txQuotaImg from 'assets/imgs/txQuota.svg';
import txRegImg from 'assets/imgs/txReg.svg';
import txRewardImg from 'assets/imgs/txReward.svg';
import txTokenImg from 'assets/imgs/txToken.svg';
import txTransImg from 'assets/imgs/txTrans.svg';
import txVoteImg from 'assets/imgs/txVote.svg';

import pagination from 'components/pagination.vue';
import tabelList from 'components/tabelList.vue';
import secTitle from 'components/secTitle';
import date from 'utils/date.js';
import { timer } from 'utils/asyncFlow';
import ellipsisAddr from 'utils/ellipsisAddr.js';
import loopTime from 'config/loopTime';

let transListInst = null;
let txImgs = [txRegImg, txRegImg, txRegImg, txRewardImg, txVoteImg, txVoteImg, txQuotaImg, txQuotaImg, txTokenImg, txTokenImg, txTransImg];

export default {
    components: {
        pagination, tabelList, secTitle
    },
    mounted() {
        this.currentPage = this.$store.state.transList.currentPage;
        this.startLoopTransList();
    },
    data() {
        let activeAccount = this.$wallet.getActiveAccount();
        let address = activeAccount.getDefaultAddr();
        return {
            acc: activeAccount,
            address, 
            currentPage: this.$store.state.transList.currentPage
        };
    },
    computed: {
        totalPage() {
            return this.$store.getters.totalPage;
        },
        pageNumber() {
            return `${this.currentPage + 1}/${this.totalPage}`;
        },
        transList() {
            let transList = this.$store.getters.transList;
            let nowList = [];

            transList.forEach((trans) => {
                let typeImg = `<img class="icon" src='${txImgs[trans.rawData.txType]}'/>`;

                let status = ['unconfirmed', 'confirms', 'confirmed'][trans.status];
                let statusClass = status === 'confirmed' ? 'green' : 
                    status === 'unconfirmed' ? 'pink': 'blue';
                let statusText = this.$t(`transList.status.${status}`) + (status === 'confirms' ? `(${trans.confirms})` : '');

                let isZero = viteWallet.BigNumber.isEqual(trans.amount, 0);
                let amount = trans.amount;
                if (!isZero) {
                    amount = trans.isSend ? ('-' + trans.amount) : ('+' + trans.amount);
                }

                nowList.push({
                    type: typeImg + this.$t(`txType.${trans.rawData.txType}`),
                    smallType: typeImg,
                    date: date(trans.timestamp, this.$i18n.locale),
                    status: `<span class="${statusClass}">${statusText}</span>`,
                    hash: trans.rawData.hash,
                    transAddr: ellipsisAddr(trans.transAddr),
                    smallTransAddr: ellipsisAddr(trans.transAddr, 6),
                    amount: `<span class="${trans.isSend ? 'red' : 'green'}">${amount}</span>`,
                    smallAmount: `<span class="${trans.isSend ? 'red' : 'green'}">${amount}</span> ` + trans.tokenSymbol,
                    tokenSymbol: trans.tokenSymbol,
                    rawData: trans.rawData
                });
            });
            return nowList;
        },
    },
    beforeDestroy() {
        this.stopLoopTransList();
    },
    methods: {
        goDetail(trans) {
            let locale = this.$i18n.locale === 'zh' ? 'zh/' : '';
            window.open(`${process.env.viteNet}${locale}transaction/${trans.rawData.hash}`);
        },

        toPage(pageNumber) {
            let pageIndex = pageNumber - 1;
            if ((pageIndex >= this.totalPage && pageIndex) || pageIndex < 0) {
                return;
            }

            this.currentPage = pageIndex;
            this.stopLoopTransList();

            this.fetchTransList(this.currentPage, true).then((data)=>{
                data && this.$refs.tableContent && (this.$refs.tableContent.scrollTop = 0);
                this.startLoopTransList();
            }).catch(()=>{
                this.startLoopTransList();
            });
        },

        startLoopTransList() {
            this.stopLoopTransList();
            transListInst = new timer(()=>{
                return this.fetchTransList(this.currentPage);
            }, loopTime.ledger_getBlocks);
            transListInst.start();
        },
        stopLoopTransList() {
            transListInst && transListInst.stop();
            transListInst = null;
        },

        fetchTransList(pageIndex) {
            return this.$store.dispatch('fetchTransList', {
                address: this.address,
                pageIndex
            });
        }
    }
};
</script>

<style lang="scss" scoped>
@import "~assets/scss/vars.scss";

.trans-list-wrapper {
    display: flex;
    flex-direction: column;
    box-sizing: border-box;
    padding: 40px;
    height: 100%;
    .title {
        margin-bottom: 40px;
    }
    .trans-list-content {
        overflow: auto;
        flex: 1;
    }
}
.small-trans {
    display: none;
}

@media only screen and (max-width: 550px) {
    .trans-list-wrapper {
        padding: 15px;
    }
}

@media only screen and (max-width: 500px) {
    .trans-list-wrapper .title {
        margin-bottom: 15px;
    }
    .big-trans {
        display: none;
    }
    .small-trans {
        display: flex;
    }
}
</style>

<style lang="scss">
@import "~assets/scss/vars.scss";

.tType {    
    min-width: 230px;
    width: 15%;
}
.status {
    min-width: 120px;
    width: 10%;
}
.time {
    min-width: 200px;
    width: 20%;
}
.address {
    min-width: 240px;
    width: 25%;
}
.sum {
    width: 14%;
    min-width: 150px;
}
.token {
    min-width: 70px;
}
.pink {
    font-family: $font-bold, arial, sans-serif;
    color: #EA60AC;
}
.blue {
    font-family: $font-bold, arial, sans-serif;
    color: #007AFF;
}
.green {
    font-family: $font-bold, arial, sans-serif;
    color: #5BC500;
}
.red {
    font-family: $font-bold, arial, sans-serif;
    color: #FF0008;
}
.icon {
    margin-right: 6px;
    margin-bottom: -2px;
}

@media only screen and (max-width: 500px) {    
    .small-trans.__tb{
        min-width: 0;
    }
    .tType {
        min-width: 50px;
        width: 10%;
    }
    .address {
        overflow: hidden;
        min-width: 180px;
        width: 25%;
    }
}
</style>
