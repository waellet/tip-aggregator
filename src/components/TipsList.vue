<template>
  <div>
    <div>
      <div class="overlay-loader" v-show="!client || showLoading">
        <BiggerLoader :progress="loadingProgress"></BiggerLoader>
      </div>
    </div>
    <div id="app-content">
      <h2>tip explorer</h2>
      <div class="container">
          <ae-list v-if="tips && tips.length">
            <ae-list-item fill="neutral" v-for="(tip,index) in tips" :key="index">
                <ae-identicon class="identicon" :address="tip.sender" size="base" />
                <div class="text-left mr-auto tip-content">
                    <ae-text face="mono-xs" > {{ tip.url }} </ae-text>
                    <ae-address :value="tip.sender" length="flat" />
                    <ae-text face="mono-xs" class="transactionDate">{{ new Date(tip.received_at).toLocaleString() }}</ae-text>
                    <ae-text face="mono-xs"> {{ tip.note }} </ae-text>
                </div>
                <div>
                    <span class="balance">{{ tip.amount }}</span>
                </div>
            </ae-list-item>
        </ae-list>
    </div>
  </div>
  </div>
</template>

<script>
  import 'prismjs'
  import 'prismjs/themes/prism.css'
  import 'prismjs/components/prism-reason.min.js'
  import 'vue-prism-editor/dist/VuePrismEditor.css' // import the styles
  import PrismEditor from 'vue-prism-editor'

  import {Universal} from '@aeternity/aepp-sdk/es/ae/universal'
  import * as Crypto from '@aeternity/aepp-sdk/es/utils/crypto'
  import {AeButton, AeInput, AeLabel, AeList, AeListItem, AeCheck} from '@aeternity/aepp-components'

  import CONTRACT_TIP_ANY from '../../contracts/tip_any_basic.aes';

  import BiggerLoader from './BiggerLoader'

  export default {
    name: 'TipsList',
      components: {
        AeInput,
        AeButton,
        AeLabel,
        AeList,
        AeListItem,
        AeCheck,
        PrismEditor,
        BiggerLoader
      },
      data() {
        return {
          nodeUrl: "https://testnet.aeternity.art",
          keypair: Crypto.generateKeyPair(),
          contractCode: CONTRACT_TIP_ANY,
          contractInstance: null,
          contractAddress: 'ct_23RBxJBhNwixiiaKFJNKSig3yYKUDwC5iouaEGjQsogNvkGS3M',
          address: null,
          client: null,
          tasks: [],
          showLoading: true,
          functionAddTaskExists: false,
          addTodoText: "",
          loadingProgress: "",
          allErrors: [],
          tips: null
        }
      },
      methods: {
        handleContractError(message) {
          this.showLoading = false;

          const matchFunctionNotFound = message.toString().match(/\.(\w*) is not a function/);
          console.log(matchFunctionNotFound);
          if (matchFunctionNotFound) message = matchFunctionNotFound[1] + " function not defined (maybe commented out?)";

          this.allErrors.push(message);
        },
        transformTasksList(list) {
          return list.map(([id, task]) => {
            return {...{id: id}, ...task};
          });
        },
        async getContractInstanceAtAddress(contractAddress) {
          try {
            this.contractInstance = await this.client.getContractInstance(this.contractCode, { contractAddress });
          } catch (e) {
            console.log(e)
          }
        },
        async getContractState() {
          try {
            const contractState = await this.contractInstance.methods.get_state();
            return contractState.decodedResult;
          } catch (e) {
            console.log(e)
          }
        },
        convertToAE(balance) {
          return +(balance / 10 ** 18).toFixed(7);
        },
        parseTips(tips) {
          let temp = [];
          tips.forEach(t => {
            let p = t[1]
            p.url = t[0][0]
            p.amount = this.convertToAE(p.amount)
            temp.push(p)
          });
          return temp;
        }
      },
      async created() {
          this.loadingProgress = "fetching tips";
          this.client = await Universal({
              url: this.nodeUrl,
              internalUrl: this.nodeUrl,
              keypair: this.keypair,
              networkId: 'ae_uat',
              compilerUrl: "https://compiler.aepps.com"
          });

          await this.getContractInstanceAtAddress(this.contractAddress);
          this.tips = await this.getContractState();
          this.tips = this.parseTips(this.tips.tips);
          this.showLoading = false;
      },
  }
</script>

<style>
  #app-content {
    max-width: 1200px;
    padding: 0 20px 20px;
  }

  #check-contract {
    margin-top: 10px;
  }

  #reset-contract {
    margin-left: 10px;
  }

  .todo-list {
    margin-top: 2rem;
  }

  .editor {
    display: block;
    max-width: 100vw;
  }

  ul {
  list-style: none;
}

  #add-todo {
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-bottom: 1rem;
  }

  #add-todo-button {
    min-width: 170px;
    margin-left: 10px;
  }

  .completed-task {
    text-decoration: line-through;
  }

  .errors div {
    padding: 0.5rem 1rem;
    background: rgba(255, 0, 0, 0.6);
    color: white;
    margin: 1rem 0;
    display: flex;
    align-items: center;
    text-align: left;
  }

  .errors div::before {
    content: "!";
    font-size: 2rem;
    font-weight: bold;
    margin-right: 1rem;
  }

  .overlay-loader {
    top: 0;
    left: 0;
    bottom: 0;
    right: 0;
    z-index: 100;
    width: 100%;
    position: absolute;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding-top: 150px;
    background-color: rgba(255, 255, 255, 0.85);
    overflow: hidden;
  }

  input.ae-input {
    box-sizing: border-box;
  }

  .ae-check-button::after {
    top: 2px !important;
    left: 7px !important;
    background-size: 1rem !important;
  }

  .non-completed-task {
    width: 100%;
    text-align: start;
  }

  #next-status-button {
    padding: 0 0.8rem;
    height: 40px;
    float: right;
  }

  .status-label {
    padding: 5px;
    border-radius: 5px;
  }

  .status-label.InProgress {
    background-color: yellow;
  }

  .status-label.ReadyForReview {
    color: white;
    background-color: blue;
  }

  .status-label.ToBeDeployed {
    background-color: limegreen;
  }

  .actions{
      width:50%;
      margin-top: 5px;
  }
  .tipWebsiteHeader  {
      margin-bottom:20px;
  }
    .ae-divider {
        background-color: #bbbbbb !important;
    }
    .domainFavicon {
        width:32px;
        margin-right:15px;
    }
    .noFavicon {
        font-size:0.8rem;
        height:32px;
        word-break: break-word;
    }
    .domainInfo {
    flex-grow:1;
    }
    
    .domain { 
        position:relative;
        cursor: pointer;
    }
    .domain:hover .full-domain{
        display:block;
    }
    .full-domain {
        display:none;
        position:absolute;
        left:0;
        right: 0;
        top: 115%;
        background: #001833;
        color: #fff;
        padding: 5px;
        border-radius: 6px;
        word-break: break-word;
        white-space: normal;
        z-index: 15;
        font-size: 0.8rem;
    }
    .full-domain:after {
        content:"";
        border: solid black;
        border-width: 0 3px 3px 0;
        display: inline-block;
        padding: 3px;
        transform: rotate(-135deg);
        -webkit-transform: rotate(-135deg);
        -ms-transform: rotate(-135deg);
        position: absolute;
        top: -4px;
        left: 23px;
        background: inherit;
    }
    h6{
        margin:0;
        word-break: break-word;
    }
    p {
        margin:0;
        font-weight:normal;
    }
    .ae-icon {
        color: #fff !important;
        width: 20px !important;
        height: 20px !important;
        font-size: 0.8rem;
        margin-right: 2px;
        border: 2px solid #dae1ea;
    }
.balance {
    color: red;
    font-size: 0.5em;
    font-weight:bold;
}
.textarea {
    min-height: 60px;
}
    .ae-badge {
        border-radius:20px;
        width:20%;
        justify-content:center;
        cursor: pointer;
        background:#e4e4e4;
    }
    .ae-badge.primary {
        background: #e4e4e4;
        color:#fff;
    }
    .ae-badge.alternative {
        background: #e4e4e4;
        color:#fff;
    }
.sendTip { 
    margin-top:25px;
}
.tipSlider {
    margin:25px 0;
}
.hideSlider {
    opacity: 0.3;
}
.amount-container{
    position:relative;
}
.hideSlider .sliderOver {
    position:absolute;
    z-index:50;
    left:0;
    right:0;
    bottom:0;
    top:0;
}
.balanceInfo {
    margin-top:15px;
}
.btn-50 {
    width:50%;
}
.ae-address {
    font-weight: bold !important;
    color:#000;
}
.tabs {
    margin-top:1rem;
}
.tabs span {
    width:49%;
}

    .balance {
        font-weight: bold;
        font-size:2rem;
        color:#000;
    }
    .balance:after {
        font-size:1rem;
        content:'AE'
    }
    small {
        font-size:.8rem;
    }
.tip-content {
    width:60%;
}
</style>
