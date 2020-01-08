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
        {{ tips }}
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
            nextStatus(status) {
                if (!status || status == 'None') return "InProgress";
                if (status === "InProgress") return "ReadyForReview";
                if (status === "ReadyForReview") return "ToBeDeployed";
                if (status === "ToBeDeployed") return "";
            },
            async setNextStatus(id, status) {
                try {
                    this.allErrors = [];
                    const nextStatus = this.nextStatus(status);

                    if (nextStatus !== "") {
                        this.showLoading = true;
                        this.loadingProgress = "calling set_task_status on contract";
                        await this.contractInstance.methods.set_task_status(id, nextStatus).catch(this.handleContractError);
                        this.loadingProgress = "calling get_tasks on contract";
                        const tasksCall = await this.contractInstance.methods.get_tasks().catch(this.handleContractError);
                        this.tasks = this.transformTasksList(tasksCall.decodedResult);
                        this.showLoading = false;
                        this.loadingProgress = "";
                    }
                } catch (e) {
                    this.handleContractError(e)
                }
            },
            async addTodo() {
                try {
                    this.allErrors = [];

                    if (this.addTodoText !== "") {
                        this.showLoading = true;
                        this.loadingProgress = "calling add_task on contract";
                        await this.contractInstance.methods.add_task(this.addTodoText).catch(this.handleContractError);
                        this.loadingProgress = "calling get_tasks on contract";
                        const tasksCall = await this.contractInstance.methods.get_tasks().catch(this.handleContractError);
                        this.tasks = this.transformTasksList(tasksCall.decodedResult);
                        this.addTodoText = "";
                        this.showLoading = false;
                        this.loadingProgress = "";
                    }
                } catch (e) {
                    this.handleContractError(e)
                }
            },
            async setTaskCompleted(id) {
                try {
                    this.allErrors = [];
                    this.showLoading = true;
                    this.loadingProgress = "calling set_task_completed on contract";
                    await this.contractInstance.methods.set_task_completed(id);
                    this.loadingProgress = "calling get_tasks on contract";
                    const tasksCall = await this.contractInstance.methods.get_tasks();
                    this.tasks = this.transformTasksList(tasksCall.decodedResult);
                    this.showLoading = false;
                    this.loadingProgress = "";
                } catch (e) {
                    this.handleContractError(e)
                }
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
                temp.push(p)
              });
              return temp;
            }
        },
        async created() {
            this.loadingProgress = "initializing sdk client";
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
</style>
