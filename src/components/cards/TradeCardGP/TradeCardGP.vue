<template>
  <BalCard class="relative card-container" :shadow="tradeCardShadow" no-border>
    <template v-slot:header>
      <div class="w-full flex items-center justify-between">
        <h4>{{ title }}</h4>
        <TradeSettingsPopover
          :context="TradeSettingsContext.trade"
          :isGasless="trading.tradeGasless.value"
        />
      </div>
    </template>
    <div>
      <TradePair
        v-model:tokenInAmount="tokenInAmount"
        v-model:tokenInAddress="tokenInAddress"
        v-model:tokenOutAmount="tokenOutAmount"
        v-model:tokenOutAddress="tokenOutAddress"
        v-model:exactIn="exactIn"
        :tradeLoading="
          trading.isBalancerTrade.value ? trading.isLoading.value : false
        "
        :effectivePriceMessage="trading.effectivePriceMessage"
        @amountChange="trading.handleAmountChange"
        class="mb-4"
      />
      <BalAlert
        v-if="error"
        class="p-3 mb-4"
        type="error"
        size="sm"
        :title="error.header"
        :description="error.body"
        :action-label="error.label"
        block
        @actionClick="handleErrorButtonClick"
      />
      <BalAlert
        v-else-if="warning"
        class="p-3 mb-4"
        type="warning"
        size="sm"
        :title="warning.header"
        :description="warning.body"
        block
      />
      <BalBtn
        v-if="trading.isLoading.value"
        loading
        disabled
        :loading-label="
          trading.isGnosisTrade.value ? $t('loadingBestPrice') : $t('loading')
        "
        block
      />
      <BalBtn
        v-else
        :label="$t('preview')"
        :disabled="tradeDisabled"
        color="gradient"
        block
        @click.prevent="handlePreviewButton"
      />
      <div
        v-if="
          !ENABLE_LEGACY_TRADE_INTERFACE &&
            trading.isGnosisSupportedOnNetwork.value
        "
        class="mt-5 text-sm flex items-center h-8"
      >
        <Transition name="fade" mode="out-in">
          <div
            v-if="trading.isGaslessTradingDisabled.value"
            class="text-secondary"
          >
            <div class="flex items-center gap-2">
              <span class="text-lg">⛽</span>
              <Transition name="fade" mode="out-in">
                <p v-if="trading.isWrap.value">
                  {{ $t('tradeToggle.wrapEth') }}
                </p>
                <p v-else-if="trading.isUnwrap.value">
                  {{ $t('tradeToggle.unwrapEth') }}
                </p>
                <p v-else>{{ $t('tradeToggle.fromEth') }}</p>
              </Transition>
            </div>
          </div>

          <div v-else>
            <div class="trade-gasless flex items-center">
              <BalTooltip
                width="64"
                :disabled="!trading.isGaslessTradingDisabled.value"
              >
                <template v-slot:activator>
                  <BalToggle
                    name="tradeGasless"
                    :checked="trading.tradeGasless.value"
                    @toggle="trading.toggleTradeGasless"
                    :disabled="trading.isGaslessTradingDisabled.value"
                  />
                </template>
                <div
                  v-text="
                    trading.isWrapUnwrapTrade.value
                      ? $t('tradeGaslessToggle.disabledTooltip.wrapUnwrap')
                      : $t('tradeGaslessToggle.disabledTooltip.eth')
                  "
                ></div>
              </BalTooltip>
              <Transition name="fade" mode="out-in">
                <span
                  v-if="trading.tradeGasless.value"
                  class="text-sm pl-2 text-gray-600 dark:text-gray-400"
                  >{{ $t('tradeToggle.tradeGasless') }}</span
                >
                <span
                  v-else
                  class="text-sm pl-2 text-gray-600 dark:text-gray-400"
                  >{{ $t('tradeToggle.tradeWithGas') }}</span
                >
              </Transition>
              <BalTooltip width="64">
                <template v-slot:activator>
                  <BalIcon
                    name="info"
                    size="xs"
                    class="text-gray-400 ml-1 flex"
                  />
                </template>
                <div v-html="$t('tradeGaslessToggle.tooltip')" />
              </BalTooltip>
            </div>
          </div>
        </Transition>
      </div>
      <TradeRoute
        v-if="alwaysShowRoutes"
        :address-in="trading.tokenIn.value.address"
        :amount-in="trading.tokenInAmountInput.value"
        :address-out="trading.tokenOut.value.address"
        :amount-out="trading.tokenOutAmountInput.value"
        :pools="trading.sor.pools.value"
        :sor-return="trading.sor.sorReturn.value"
        class="mt-4"
      />
    </div>
  </BalCard>
  <teleport to="#modal">
    <TradePreviewModalGP
      v-if="modalTradePreviewIsOpen"
      :trading="trading"
      @trade="trade"
      @close="handlePreviewModalClose"
    />
  </teleport>
</template>

<script lang="ts">
import { getAddress, isAddress } from '@ethersproject/address';
import { formatUnits } from '@ethersproject/units';
import { computed, defineComponent, onBeforeMount, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRouter } from 'vue-router';
import { useStore } from 'vuex';

import TradePreviewModalGP from '@/components/modals/TradePreviewModalGP.vue';
import TradeSettingsPopover, {
  TradeSettingsContext
} from '@/components/popovers/TradeSettingsPopover.vue';
import { ENABLE_LEGACY_TRADE_INTERFACE } from '@/composables/trade/constants';
import { useTradeState } from '@/composables/trade/useTradeState';
import useTrading from '@/composables/trade/useTrading';
import useValidation, {
  TradeValidation
} from '@/composables/trade/useValidation';
import useBreakpoints from '@/composables/useBreakpoints';
import useNumbers, { FNumFormats } from '@/composables/useNumbers';
import useTokens from '@/composables/useTokens';
import { TOKENS } from '@/constants/tokens';
import { lsGet } from '@/lib/utils';
import { WrapType } from '@/lib/utils/balancer/wrapper';
import { isRequired } from '@/lib/utils/validations';
import { ApiErrorCodes } from '@/services/gnosis/errors/OperatorError';
import useWeb3 from '@/services/web3/useWeb3';

import TradePair from '../TradeCard/TradePair.vue';
import TradeRoute from '../TradeCard/TradeRoute.vue';

export default defineComponent({
  components: {
    TradePair,
    TradePreviewModalGP,
    TradeRoute,
    TradeSettingsPopover
  },

  setup() {
    // COMPOSABLES
    const store = useStore();
    const router = useRouter();
    const { t } = useI18n();
    const { bp } = useBreakpoints();
    const { fNum2 } = useNumbers();
    const { appNetworkConfig } = useWeb3();
    const { nativeAsset } = useTokens();
    const {
      tokenInAddress,
      tokenOutAddress,
      tokenInAmount,
      tokenOutAmount,
      setTokenInAddress,
      setTokenOutAddress,
      setInitialized
    } = useTradeState();

    // DATA
    const exactIn = ref(true);
    const modalTradePreviewIsOpen = ref(false);
    const dismissedErrors = ref({
      highPriceImpact: false
    });
    const alwaysShowRoutes = lsGet('alwaysShowRoutes', false);

    const tradeCardShadow = computed(() => {
      switch (bp.value) {
        case 'xs':
          return 'none';
        case 'sm':
          return 'lg';
        default:
          return 'xl';
      }
    });

    const trading = useTrading(
      exactIn,
      tokenInAddress,
      tokenInAmount,
      tokenOutAddress,
      tokenOutAmount
    );

    // COMPUTED
    const { errorMessage } = useValidation(
      tokenInAddress,
      tokenInAmount,
      tokenOutAddress,
      tokenOutAmount
    );

    const isHighPriceImpact = computed(
      () =>
        trading.sor.validationErrors.value.highPriceImpact &&
        !dismissedErrors.value.highPriceImpact
    );

    const tradeDisabled = computed(() => {
      const hasValidationError = errorMessage.value !== TradeValidation.VALID;
      const hasGnosisErrors =
        trading.isGnosisTrade.value && trading.gnosis.hasValidationError.value;
      const hasBalancerErrors =
        trading.isBalancerTrade.value && isHighPriceImpact.value;

      return hasValidationError || hasGnosisErrors || hasBalancerErrors;
    });

    const title = computed(() => {
      if (trading.wrapType.value === WrapType.Wrap) {
        return `${t('wrap')} ${trading.tokenIn.value.symbol}`;
      }
      if (trading.wrapType.value === WrapType.Unwrap) {
        return `${t('unwrap')} ${trading.tokenOut.value.symbol}`;
      }
      return t('trade');
    });

    const error = computed(() => {
      if (trading.isBalancerTrade.value) {
        if (errorMessage.value === TradeValidation.NO_LIQUIDITY) {
          return {
            header: t('insufficientLiquidity'),
            body: t('insufficientLiquidityDetailed')
          };
        }
      }

      if (trading.isGnosisTrade.value) {
        if (trading.gnosis.validationError.value != null) {
          const validationError = trading.gnosis.validationError.value;

          if (validationError === ApiErrorCodes.SellAmountDoesNotCoverFee) {
            return {
              header: t('gnosisErrors.lowAmount.header'),
              body: t('gnosisErrors.lowAmount.body')
            };
          } else if (validationError === ApiErrorCodes.PriceExceedsBalance) {
            return {
              header: t('gnosisErrors.lowBalance.header', [
                trading.tokenIn.value.symbol
              ]),
              body: t('gnosisErrors.lowBalance.body', [
                trading.tokenIn.value.symbol,
                fNum2(
                  formatUnits(
                    trading.getQuote().maximumInAmount,
                    trading.tokenIn.value.decimals
                  ),
                  FNumFormats.token
                ),
                fNum2(trading.slippageBufferRate.value, FNumFormats.percent)
              ])
            };
          } else if (validationError === ApiErrorCodes.NoLiquidity) {
            return {
              header: t('gnosisErrors.noLiquidity.header', [
                trading.tokenIn.value.symbol
              ]),
              body: t('gnosisErrors.noLiquidity.body')
            };
          } else {
            return {
              header: t('gnosisErrors.genericError.header'),
              body: trading.gnosis.validationError.value
            };
          }
        }
      } else if (trading.isBalancerTrade.value) {
        if (isHighPriceImpact.value) {
          return {
            header: t('highPriceImpact'),
            body: t('highPriceImpactDetailed'),
            label: t('accept')
          };
        }
      }

      return undefined;
    });

    const warning = computed(() => {
      if (trading.isGnosisTrade.value) {
        if (trading.gnosis.warnings.value.highFees) {
          return {
            header: t('gnosisWarnings.highFees.header'),
            body: t('gnosisWarnings.highFees.body')
          };
        }
      }

      return undefined;
    });

    // METHODS
    function trade() {
      trading.trade(() => {
        trading.resetAmounts();
        modalTradePreviewIsOpen.value = false;
      });
    }

    function handleErrorButtonClick() {
      if (trading.sor.validationErrors.value.highPriceImpact) {
        dismissedErrors.value.highPriceImpact = true;
      }
    }

    async function populateInitialTokens(): Promise<void> {
      let assetIn = router.currentRoute.value.params.assetIn as string;

      if (assetIn === nativeAsset.deeplinkId) {
        assetIn = nativeAsset.address;
      } else if (isAddress(assetIn)) {
        assetIn = getAddress(assetIn);
      }

      let assetOut = router.currentRoute.value.params.assetOut as string;

      if (assetOut === nativeAsset.deeplinkId) {
        assetOut = nativeAsset.address;
      } else if (isAddress(assetOut)) {
        assetOut = getAddress(assetOut);
      }
      setTokenInAddress(assetIn || store.state.trade.inputAsset);
      setTokenOutAddress(assetOut || store.state.trade.outputAsset);
    }

    function switchToWETH() {
      tokenInAddress.value = appNetworkConfig.addresses.weth;
    }

    function handlePreviewButton() {
      trading.resetSubmissionError();

      modalTradePreviewIsOpen.value = true;
    }

    function handlePreviewModalClose() {
      trading.resetSubmissionError();

      modalTradePreviewIsOpen.value = false;
    }

    // INIT
    onBeforeMount(() => {
      populateInitialTokens();
      setInitialized(true);
    });

    return {
      // constants
      TOKENS,
      ENABLE_LEGACY_TRADE_INTERFACE,
      // context
      TradeSettingsContext,

      // data
      tokenInAddress,
      tokenInAmount,
      tokenOutAddress,
      tokenOutAmount,
      modalTradePreviewIsOpen,
      alwaysShowRoutes,
      exactIn,
      trading,

      // computed
      title,
      error,
      warning,
      errorMessage,
      isRequired,
      tradeDisabled,
      tradeCardShadow,
      handlePreviewButton,
      handlePreviewModalClose,

      // methods
      trade,
      switchToWETH,
      handleErrorButtonClick
    };
  }
});
</script>
<style scoped>
/* This is needed because the trade settings popover overflows */
.card-container {
  overflow: unset;
}

.trade-gasless :deep(.bal-toggle) {
  width: 3rem;
}
.gas-symbol {
  @apply h-8 w-8 rounded-full flex items-center justify-center text-lg bg-gray-50 dark:bg-gray-800;
}
.gas-symbol:before {
  content: '⛽';
}
.signature-symbol:before {
  content: '✍️';
}
</style>
