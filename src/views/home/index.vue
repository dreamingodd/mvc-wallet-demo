<template>
  <div class="home-container pubBack">
    <el-row>
      <el-input placeholder="请输入地址" v-model="balance">
        <template slot="prepend">查询余额</template>
        <el-button slot="append" icon="el-icon-search" @click="getBalance"></el-button>
      </el-input>
      <el-col :span="24">
        <div class="balance-con">余额：<span>{{balanceCoin}} （eth）</span></div>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="8">
        <el-input placeholder="请输入地址" v-model="transferFrom">
          <template slot="prepend">转出</template>
        </el-input>
      </el-col>
      <el-col :span="8">
        <el-input placeholder="请输入地址" v-model="transferTo">
          <template slot="prepend">转入</template>
        </el-input>
      </el-col>
      <el-col :span="8">
        <el-input placeholder="请输入地址" type="number" v-model="transferCount">
          <template slot="prepend">转账数量</template>
          <el-button slot="append" @click="getTransfer">转账</el-button>
        </el-input>
      </el-col>
      <el-col :span="24">
        <div class="balance-con">TXhash值：<span>{{orderHash}}</span></div>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="24">
        <el-input placeholder="请输入地址/hash" v-model="orderHash">
          <template slot="prepend">查询订单信息</template>
          <el-button slot="append" icon="el-icon-search" @click="getTransferInfo"></el-button>
        </el-input>
      </el-col>
      <el-col :span="24">
        <el-card class="box-card">
          <span v-if="!tableData.blockHash">暂无信息</span>
          <div v-for="(v,k) in tableData" :key="k">
            <div>
              <b>{{k}}:</b><span>{{v}}</span>
            </div>
          </div>
        </el-card>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="24">
        <el-input placeholder="请输入秘钥" v-model="privateKey">
          <template slot="prepend">导入钱包：输入秘钥</template>
          <el-button slot="append" icon="el-icon-search" @click="getPrivateWallet"></el-button>
        </el-input>
      </el-col>
      <el-col :span="24">
        <el-card class="box-card">
          <div class="balance-con" v-if="keyResult">你的地址：<span>{{keyResult}}</span></div>
          <div class="balance-con" v-if="keyResult === null">秘钥文件已存在</div>
        </el-card>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="24">
          <el-button @click="exportKeyFile">到处key文件</el-button>
      </el-col>
    </el-row>
  </div>
</template>
<script type='text/ecmascript-6'>
  let encrypt = new window.JSEncrypt();
  export default {
    data() {
      return {
        balance: '',
        balanceCoin: '0.00',
        transferCount: 0.001,
        keyResult: '0x5D748f8C84f90348fB3A4AE2F8B815010a91CF73',
        orderHash: '',
        privateKey: '1bbd568c95b4bc2fb75056921b781adc66dad3471d25d90e002849c46b8ef400',
        pKey: '',
        transferFrom: '0x58f103AdABe28D60febfB2fB732FEf8C7aCDbDa3',
        transferTo: '0x98fd7a69b3550f3d24998b32254191f701b0afb1',
        tableData: {},
        blockId: 'latest'
      };
    },
    mounted: function () {
      this.$store.dispatch('getPublicKey').then((res) => {
        this.pKey = res;
        encrypt.setPublicKey(this.pKey);
      }).catch((err) => {
        this.$message.error(err);
      });
    },
    components: {},

    methods: {
      getBalance() {
        if (this.balance.replace(/\s/ig, '')) {
          this.$store.dispatch('getBalance', {address: this.balance, blockId: this.blockId}).then((res) => {
            this.balanceCoin = res;
          }).catch((err) => {
            this.$message.error(err);
          });
        } else {
          this.balanceCoin = '0.00';
        }
      },
      getTransfer() {
        if (this.transferFrom.replace(/\s/ig, '') && this.transferTo.replace(/\s/ig, '')) {
          this.getPassword();
        } else {
          this.$message.error('转账地址不能为空');
        }
      },
      getPassword() {
        if (!Number(this.transferCount) || Number(this.transferCount) <= 0) {
          this.$message.error('转账数量不能为0或负数');
          return;
        }
        this.$prompt('请转账秘钥', '提示', {
          confirmButtonText: '确定',
          inputType: 'password',
          cancelButtonText: '取消'
        }).then(({value}) => {
          let that = this;
          let encrypted = encrypt.encrypt(value);
          let tx = {
            'pass': encrypted,
            'from': that.transferFrom,
            'to': that.transferTo,
            'value': that.transferCount
          };
          this.$store.dispatch('sendTransaction', tx).then((res) => {
            this.orderHash = res.result;
          }).catch((err) => {
            this.$message.error(err);
          });
        });
      },
      getTransferInfo() {
        if (!this.orderHash) {
          this.tableData = {};
          return;
        }
        this.$store.dispatch('getTransactionBtHash', {transactionHash: this.orderHash}).then((res) => {
          let rej = res.result;
          if (typeof rej === 'object') {
            rej.gas = window.web3.fromWei(rej.gas);
            rej.gasPrice = window.web3.fromWei(Number(rej.gasPrice));
            rej.value = window.web3.fromWei(Number(rej.value)) + 'ether';
            this.tableData = rej;
          }
        }).catch((err) => {
          this.$message.error(err);
        });
      },
      getPrivateWallet() {
        this.$prompt('你的钱包被加密，请输入密码：', '提示', {
          confirmButtonText: '确定',
          inputType: 'password',
          cancelButtonText: '取消'
        }).then(({value}) => {
          let encrypted = encrypt.encrypt(value);
          let encrypted1 = encrypt.encrypt(this.privateKey);
          this.$store.dispatch('getKeyFile', {keydata: encrypted1, passphrase: encrypted}).then((res) => {
            this.keyResult = res.result;
          }).catch((err) => {
            this.$message.error(err);
          });
        });
      },
      exportKeyFile() {

      }
    }
  };
</script>
<style lang="stylus" rel="stylesheet/stylus">
  @import './index.styl';
  .el-row
    margin-bottom: 29px;

  .balance-con
    text-align: left;
    background #fff;
    padding: 10px 0;
    text-indent: 20px;

  .box-card
    text-align: left;

    & b
      display: inline-block;
      width: 200px;
      padding: 5px 0;

</style>
