<script setup lang="ts">
import { ref, computed, watch } from 'vue';
import Selector from './Selector.vue';
import { useWeb3ModalProvider } from '@web3modal/ethers/vue';
import { useNetwork } from '../../../providers/network';
import { useController } from '../../../utils/RewardFaucetController';
import { useVeSystem } from '../../../providers/veSystem';
import { toBigInt, ethers } from 'ethers';
import { getSelectorTokenItems } from '../../../utils';
import { addRewardsIntoNWeeksHint } from '../../../constants/hints';
import Tooltip from './Tooltip.vue';

const { walletProvider } = useWeb3ModalProvider();
const { selected: veSystem } = useVeSystem();
const { network } = useNetwork();
const { tokenAllowance, approveToken, depositEqualWeeksPeriod } = useController(
  {
    walletProvider,
    network,
    veSystem,
  }
);

const token = ref<string>('');
const decimals = ref<string>('');
const inputAmount = ref<string>('');
const weeksInput = ref<string>('');
const allowance = ref<bigint>(toBigInt(0));
const isLoading = ref<boolean>(false);

const amount = computed<bigint>(() => {
  if (inputAmount.value === '') return toBigInt(0);

  return ethers.parseUnits(inputAmount.value.toString(), decimals.value);
});

const weeks = computed<bigint>(() => {
  if (weeksInput.value === '') return toBigInt(0);

  return toBigInt(weeksInput.value);
});

const isAllowanceEnough = computed<boolean>(
  () => allowance.value >= amount.value
);

const showApprove = computed<boolean>(
  () =>
    inputAmount.value !== '' && token.value !== '' && !isAllowanceEnough.value
);

const fetchAllowance = async () => {
  const tokenAllowanceResult = await tokenAllowance.value?.(token.value);

  console.log('allowance: ', tokenAllowanceResult);

  allowance.value = tokenAllowanceResult || toBigInt(0);
};

watch(token, async value => {
  if (value === '') return;

  await fetchAllowance();
});

// set token address and decimals
const handleSelectToken = (address: string) => {
  if (!veSystem.value) return;

  const rewardToken = veSystem.value.rewardDistributor.rewardTokens.find(
    rt => rt.address === address
  );

  if (!rewardToken) return;

  token.value = address;

  decimals.value = rewardToken.decimals;
};

const handleSubmit = async () => {
  await depositEqualWeeksPeriod.value?.(
    { token: token.value, amount: amount.value, weeks: weeks.value },
    {
      onPrompt: () => {
        console.log('prompt');
        isLoading.value = true;
      },
      onSubmitted: ({ tx }) => {
        console.log('submitted', tx);
      },
      onSuccess: ({ receipt }) => {
        console.log('success', receipt);
        isLoading.value = false;
        clearForm();
      },
      onError: err => {
        console.log('err', err);
        isLoading.value = false;
        clearForm();
      },
    }
  );
};

const clearForm = () => {
  inputAmount.value = '';
  token.value = '';
  weeksInput.value = '';
};

const handleApprove = async () => {
  await approveToken.value?.(
    { token: token.value, amount: amount.value },
    {
      onPrompt: () => {
        console.log('prompt');
        isLoading.value = true;
      },
      onSubmitted: ({ tx }) => {
        console.log('submitted', tx);
      },
      onSuccess: async ({ receipt }) => {
        console.log('success', receipt);
        isLoading.value = false;
        await fetchAllowance();
      },
      onError: err => {
        console.log('err', err);
        isLoading.value = false;
      },
    }
  );
};

const tokens = computed<[string, string][]>(() => {
  if (veSystem.value === undefined) return [];

  return getSelectorTokenItems(veSystem.value);
});
</script>

<template>
  <div key="nWeek" class="item-row">
    <p class="item-name">
      Add Rewards into N Weeks
      <Tooltip
        :text="addRewardsIntoNWeeksHint"
        :width="350"
        position="left-bottom"
        :fontSize="12"
      >
        <svg width="16" height="16" class="icon">
          <use href="/images/exclamation-circle.svg#icon"></use>
        </svg>
      </Tooltip>
    </p>
    <div class="item-action">
      <Selector
        :items="tokens"
        :selected="token"
        prompt="Select Token"
        :onChange="handleSelectToken"
      />
      <input
        v-model="inputAmount"
        placeholder="Amount"
        type="number"
        class="input-amount"
      />
      <div class="input-group weeks-container">
        <p class="title-input">weeks</p>
        <input
          v-model="weeksInput"
          placeholder="10"
          type="number"
          class="input"
        />
      </div>
      <button
        v-show="!showApprove"
        :disabled="
          token === '' || inputAmount === '' || weeksInput === '' || isLoading
        "
        class="submit-button"
        @click="handleSubmit"
      >
        Add
      </button>
      <button
        v-show="showApprove"
        :disabled="isLoading"
        class="submit-button"
        @click="handleApprove"
      >
        Approve
      </button>
    </div>
  </div>
</template>

<style scoped>
input[type='number']::-webkit-inner-spin-button,
input[type='number']::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input[type='number'] {
  -moz-appearance: textfield;
  appearance: textfield;
}
.item-row {
  display: flex;
  width: 100%;
  align-items: center;
  justify-content: space-between;
  max-width: 700px;
  height: 45px;
  gap: 10px;
}

.item-row .item-name {
  width: 35%;
  max-width: 350px;
  font-size: 14px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 5px;
}

.item-row .item-name .icon {
  fill: #2c3e50;
}

.dark .item-row .item-name .icon {
  fill: #ffffff;
}

.item-row .item-action {
  display: flex;
  width: 65%;
  align-items: center;
  height: 100%;
  gap: 10px;
}

.item-row .input-group {
  height: 100%;
}

.item-row .input-group.weeks-container {
  width: 50px;
  position: relative;
}

.item-row .input-group .input,
.item-row .item-action .input-amount {
  background-color: transparent;
  border: 1px solid #e2e8f0;
  position: relative;
  border-radius: 6px;
  height: 100%;
  width: 100%;
  padding-inline: 20px;
  font-size: 14px;
  outline: none;
  display: flex;
  align-items: center;
}

.item-row .input-group.weeks-container .title-input {
  position: absolute;
  font-size: 11px;
  margin: 0;
  top: 1px;
}

.item-row .input-group.weeks-container .title-input {
  left: 9px;
}

.item-row .item-action .input-amount {
  width: 70px;
  padding-inline: 10px;
}

.item-row .item-action .input-group.weeks-container .input {
  width: 50px;
  padding-inline: 10px;
  padding-top: 10px;
}

.dark .item-row .input-group .input,
.dark .item-row .item-action .input-amount {
  border: 1px solid #3e4c5a;
}

.item-row .input-group .input:focus,
.dark .item-row .item-action .input-amount:focus {
  border: 1px solid #384aff;
}

.submit-button {
  min-width: 75px;
  height: 45px;
  background-color: #eaf0f6;
  border-radius: 6px;
  cursor: pointer;
  align-self: flex-end;
  font-weight: 600;
  font-size: 14px;
  box-shadow: none;
  border: none;
}

.dark .submit-button {
  background-color: #384aff;
}
.submit-button:disabled {
  background-color: rgba(56, 74, 255, 0.2);
  cursor: not-allowed;
}
</style>
