<template>
  <div>
    <a-card 
      class="ant-card-shadow" 
      :loading="state.loading" 
      :tab-list="tabList"
      :active-tab-key="key"
      @tabChange="onTabChange"
    >
      <template #title>
        <h3>
          {{state.data.title}}
          <span style="float:right">
            You Already Payment {{state.myAmount}} Eth
            <a-button type="primary" v-if="new Date(state.data.endTime * 1000) > new Date() && state.data.success == false" @click="openModal">I want to purchase</a-button>
            <a-button type="danger" v-if="!state.data.success && state.myAmount != 0" @click="returnM">Refund</a-button>
          </span>
        </h3>
      </template>
      <a-descriptions bordered v-if="key==='info'">
        <a-descriptions-item label="Project title" :span="2">
          {{state.data.title}}
        </a-descriptions-item>
        <a-descriptions-item label="Seller" :span="2">
          {{state.data.initiator}}
        </a-descriptions-item>
        <a-descriptions-item label="Deadline date" :span="2">
           {{new Date(state.data.endTime * 1000).toLocaleString()}}
        </a-descriptions-item>
        <a-descriptions-item label="Current state">
          <a-tag color="success" v-if="state.data.success === true">
            <template #icon>
              <check-circle-outlined />
            </template>
            Purchase success
          </a-tag>
          <a-tag color="processing" v-else-if="new Date(state.data.endTime * 1000) > new Date()" >
            <template #icon>
              <sync-outlined :spin="true" />
            </template>
            Purchase
          </a-tag>
          <a-tag color="error" v-else>
            <template #icon>
              <close-circle-outlined />
            </template>
            Purchase failure
          </a-tag>
        </a-descriptions-item>
        <a-descriptions-item label="Product price">
          <a-statistic :value="state.data.goal">
            <template #suffix>
              Eth
            </template>
          </a-statistic>
        </a-descriptions-item>
        <a-descriptions-item label="Expected price">
          <a-statistic :value="state.data.amount">
            <template #suffix>
              Eth
            </template>
          </a-statistic>
        </a-descriptions-item>
        <a-descriptions-item label="Progressing">
          <a-progress type="circle" :percent="Math.round(state.data.success ? 100 : state.data.amount * 100 / state.data.goal)" :width="80" />
        </a-descriptions-item>
        <a-descriptions-item label="Introduction">
          {{state.data.info}}
        </a-descriptions-item>
		<a-descriptions-item label="imgs">
		  <a-image
		      :width="100"
		      :height="100"
		      :src="state.imgUrl"
		    />
		</a-descriptions-item>
      </a-descriptions>

      <Use v-if="key==='use'" :id="id" :data="state.data" :amount="state.myAmount"></Use>

    </a-card>

    <Modal v-model:visible="isOpen">
      <a-card style="width: 600px; margin: 0 2em;" :body-style="{ overflowY: 'auto', maxHeight: '600px' }">
        <template #title><h3 style="text-align: center">Purchase</h3></template>
        <create-form :model="model" :form="form" :fields="fields" />
      </a-card>
    </Modal>

    <Modal v-model:visible="isOpenVerify">
      <a-card style="width: 600px; margin: 0 2em;" :body-style="{ overflowY: 'auto', maxHeight: '600px' }">
        <a-row>
          <a-col :span="3"></a-col>
          <a-col :span="3">
            <MobileTwoTone :style="{fontSize: '5em', color: '#08c'}"/>
          </a-col>
          <a-col :span="18">
            <span style="font-size: 1.2rem">Please fill in the current verification code</span>
          </a-col>
        </a-row>
        <a-row class="verify-code">
          <a-col :span="24" @click="focusInput">
            <span>{{inputValue[0]}}</span>
            <span>{{inputValue[1]}}</span>
            <span>{{inputValue[2]}}</span>
            <span>{{inputValue[3]}}</span>
            <span>{{inputValue[4]}}</span>
            <span>{{inputValue[5]}}</span>
          </a-col>
        </a-row>
        <template #title><h3 style="text-align: center">Please Enter the verification code</h3></template>
        <input ref="inputCode" v-model="inputValue" type="text" maxlength="6" style="outline: none;width:100%;height:1px;border:0;background-color: transparent;color:transparent;">
        <a-row>
          <a-col style="text-align: center" :span="24">
            <a-button style="margin-left: 2rem" type="primary" @click="verify">Verify</a-button>
          </a-col>
        </a-row>

      </a-card>
    </Modal>
  </div>
</template>

<script lang="ts">
import {defineComponent, ref, shallowRef, reactive, computed, h, nextTick} from 'vue';
import { getOneFunding, Funding, getAccount, getMyFundingAmount, contribute, returnMoney, addListener } from '../api/contract'
import { useRoute } from 'vue-router'
import { message, notification } from 'ant-design-vue';
import { CheckCircleOutlined, SyncOutlined, CloseCircleOutlined } from '@ant-design/icons-vue'
import {  MobileTwoTone } from '@ant-design/icons-vue';
import Modal from '../components/base/modal.vue'
import CreateForm from '../components/base/createForm.vue'
import Use from '../components/Use.vue'
import { Model, Fields, Form } from '../type/form'
import DataUtil from '@/components/Datautil'

const column = [
  {
    dataIndex: ''
  }
]

const tabList = [
  {
    key: 'info',
    tab: 'Product Description',
  },
  {
    key: 'use',
    tab: 'Use request',
  },
];

export default defineComponent({
  name: 'Funding',
  components: { Modal, CreateForm, MobileTwoTone, CheckCircleOutlined, CloseCircleOutlined, Use },
  setup() {
    const route = useRoute();
    const id = parseInt(route.params.id as string);
    const account = ref('');

    const state = reactive<{data: Funding | {}, loading: boolean, myAmount: number,imgUrl:string}>({
      data: {},
      loading: true,
      myAmount: 0,
	  imgUrl: ''
    })

    const codeValue = ref('');
    const verifyText = ref(verifyCode());

    function verifyCode(min: number = 100000 , max: number = 999999): number {
      const value = Math.floor(min + Math.random() * (max - min))
      codeValue.value = value.toString()
      return value
    }

    const isOpen = ref(false);
    const isOpenVerify = ref(false);
    function openModal() { isOpen.value = true }
    function closeModal() { isOpen.value = false }

    function openVerifyModal() {
      isOpenVerify.value = true;
      setTimeout(function () {
        alert('Your OTP is:' + codeValue.value)
      },300)
    }
    function closeVerifyModal() { isOpenVerify.value = false }

  
    let inputValue = ref('')
    const inputCode = shallowRef<HTMLInputElement>()

    function focusInput() {
      nextTick(()=> {
        inputCode.value?.focus()
      })
    }

    async function verify() {
      if(codeValue.value == inputValue.value) {
        closeVerifyModal();
        await contribute(id, model.value);
        message.success('Purchase success')
        fetchData();
        closeModal();
        verifyText.value = verifyCode()
        inputValue.value = ''
      } else {
        verifyText.value = verifyCode()
        inputValue.value = ''
        message.error('Verify failure')
        setTimeout(function () {
          alert('Your OTP is:' + codeValue.value)
        },1000)
      }
    }

    const model = reactive<Model>({
      value: 1
    })
    const fieldsVerify = reactive<Fields>({
      value: {
        type: 'input',
        label: 'Verification code'
      }
    })

    const fields = reactive<Fields>({
      value: {
        type: 'number',
        min: 1,
        label: 'Amount'
      }
    })

    const form = reactive<Form>({
      submitHint: 'Purchase',
      cancelHint: 'Cancel',
      cancel: () => {
        closeModal();
      },
      finish: async () => {
        try {
          openVerifyModal()

        } catch (e) {
          message.error('Purchase failure')
        }
      }
    })

    async function returnM() {
      try {
        await returnMoney(id);
        message.success('Successful refund');
        fetchData()
      } catch(e) {
        message.error('Refund failure')
      }
    }

   
    const key = ref('info');
    const onTabChange = (k : 'use' | 'info') => {
      key.value = k;
    }

   
    async function fetchData() {
      state.loading = true;
      try {
        [state.data, state.myAmount] = await Promise.all([getOneFunding(id), getMyFundingAmount(id)]);
        state.loading = false;
        //@ts-ignore
        fields.value.max = state.data.goal - state.data.amount;
		console.log(state.data)
		//@ts-ignore
		state.imgUrl = DataUtil.displayImgUrl(state.data.imgs)
      } catch (e) {
        console.log(e);
        message.error('Failed to get details');
      }
    }

    addListener(fetchData)

    getAccount().then(res => account.value = res)
    fetchData();

    return { inputCode, inputValue,verify,verifyText, isOpenVerify, focusInput, state, account, codeValue,isOpen, openModal, form, model, fields, fieldsVerify, tabList, key, onTabChange, id, returnM}
  }
});
</script>

<style>
	.ant-image-img {
		height: 100%;
	}
</style>