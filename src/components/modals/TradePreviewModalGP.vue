<template>
  <BalModal show @close="onClose" :title="labels.modalTitle">
    <div>
      <BalCard noPad class="relative mb-6 overflow-auto">
        <template v-slot:header>
          <div
            class="w-full p-3 border-b bg-gray-50 rounded-t-lg text-sm dark:border-gray-800 dark:bg-gray-800"
          >
            <span>
              {{ $t('effectivePrice') }}
              {{
                trading.exactIn.value
                  ? trading.effectivePriceMessage.value.tokenIn
                  : trading.effectivePriceMessage.value.tokenOut
              }}
            </span>
          </div>
        </template>
        <div>
          <div
            class="p-3 border-gray-100 border-b relative dark:border-gray-900"
          >
            <div class="flex items-center">
              <div class="mr-3">
                <BalAsset :address="trading.tokenIn.value.address" :size="36" />
              </div>
              <div>
                <div class="font-medium">
                  {{
                    fNum2(trading.tokenInAmountInput.value, FNumFormats.token)
                  }}
                  {{ trading.tokenIn.value.symbol }}
                </div>
                <div class="text-secondary text-sm">
                  {{ tokenInFiatValue }}
                </div>
              </div>
            </div>
          </div>
          <div class="arrow-down">
            <ArrowDownIcon />
          </div>
          <div class="p-3">
            <div class="flex items-center">
              <div class="mr-3">
                <BalAsset
                  :address="trading.tokenOut.value.address"
                  :size="36"
                />
              </div>
              <div>
                <div class="font-medium">
                  {{
                    fNum2(trading.tokenOutAmountInput.value, FNumFormats.token)
                  }}
                  {{ trading.tokenOut.value.symbol }}
                </div>
                <div class="text-secondary text-sm">
                  {{ tokenOutFiatValue }}
                  <span
                    v-if="
                      trading.isBalancerTrade.value ||
                        trading.isWrapUnwrapTrade.value
                    "
                  >
                    / {{ $t('priceImpact') }}:
                    {{
                      fNum2(trading.sor.priceImpact.value, FNumFormats.percent)
                    }}
                  </span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </BalCard>
      <BalCard noPad shadow="none" class="mb-3">
        <template v-slot:header>
          <div
            class="p-3 flex w-full items-center justify-between border-b dark:border-gray-900"
          >
            <div class="font-semibold">
              {{ labels.tradeSummary.title }}
            </div>
            <div class="flex divide-x dark:divide-gray-500 text-xs uppercase">
              <div
                :class="[
                  'pr-2 cursor-pointer font-medium',
                  { 'text-blue-600': !showSummaryInFiat }
                ]"
                @click="showSummaryInFiat = false"
              >
                {{ $t('tokens') }}
              </div>
              <div
                :class="[
                  'pl-2 cursor-pointer font-medium uppercase',
                  { 'text-blue-600': showSummaryInFiat }
                ]"
                @click="showSummaryInFiat = true"
              >
                {{ FiatCurrency.usd }}
              </div>
            </div>
          </div>
        </template>
        <div class="p-3 text-sm">
          <div class="summary-item-row">
            <div>
              {{ labels.tradeSummary.totalBeforeFees }}
            </div>
            <div v-html="summary.amountBeforeFees" />
          </div>
          <div class="summary-item-row" v-if="trading.isGnosisTrade.value">
            <div>{{ $t('tradeSummary.gasCosts') }}</div>
            <div class="text-green-400">-{{ zeroFee }}</div>
          </div>
          <div class="summary-item-row">
            <div>{{ labels.tradeSummary.tradeFees }}</div>
            <div
              v-html="
                trading.isWrapUnwrapTrade.value
                  ? zeroFee
                  : trading.isGnosisTrade.value
                  ? trading.exactIn.value
                    ? `-${summary.tradeFees}`
                    : `+${summary.tradeFees}`
                  : summary.tradeFees
              "
            />
          </div>
        </div>
        <template v-slot:footer>
          <div
            class="w-full p-3 rounded-b-lg bg-white text-sm dark:bg-gray-800"
          >
            <div class="summary-item-row font-medium">
              <div class="w-64">
                {{ labels.tradeSummary.totalAfterFees }}
              </div>
              <div v-html="summary.totalWithoutSlippage" />
            </div>
            <div class="summary-item-row text-secondary">
              <div class="w-64">
                {{ labels.tradeSummary.totalWithSlippage }}
              </div>
              <div
                v-html="
                  trading.isWrapUnwrapTrade.value
                    ? ''
                    : summary.totalWithSlippage
                "
              />
            </div>
          </div>
        </template>
      </BalCard>
      <BalAlert
        v-if="showPriceUpdateError"
        class="p-3 mb-4"
        type="error"
        size="md"
        :title="$t('priceUpdatedAlert.title')"
        :description="
          $t('priceUpdatedAlert.description', [
            fNum2(PRICE_UPDATE_THRESHOLD, FNumFormats.percent)
          ])
        "
        :action-label="$t('priceUpdatedAlert.actionLabel')"
        block
        @actionClick="cofirmPriceUpdate"
      />
      <div
        v-if="totalRequiredTransactions > 1"
        class="flex my-5 justify-center items-center"
      >
        <template v-if="showGnosisRelayerApprovalStep">
          <BalTooltip :disabled="!requiresGnosisRelayerApproval" width="64">
            <template v-slot:activator>
              <div
                :class="[
                  'step',
                  {
                    'step-active':
                      activeTransactionType === 'gnosisRelayerApproval',
                    'step-approved': !requiresGnosisRelayerApproval
                  }
                ]"
              >
                <BalIcon
                  v-if="!requiresGnosisRelayerApproval"
                  name="check"
                  class="text-green-500"
                />
                <template v-else>1</template>
              </div>
            </template>
            <div>
              <div class="mb-2 font-semibold">
                <div>
                  {{
                    $t(
                      'tradeSummary.transactionTypesTooltips.gnosisRelayerApproval.title'
                    )
                  }}
                </div>
              </div>
              <div>
                {{
                  $t(
                    'tradeSummary.transactionTypesTooltips.gnosisRelayerApproval.content'
                  )
                }}
              </div>
            </div>
          </BalTooltip>
          <div class="step-seperator" />
        </template>
        <template v-else-if="showLidoRelayerApprovalStep">
          <BalTooltip :disabled="!requiresLidoRelayerApproval" width="64">
            <template v-slot:activator>
              <div
                :class="[
                  'step',
                  {
                    'step-active':
                      activeTransactionType === 'lidoRelayerApproval',
                    'step-approved': !requiresLidoRelayerApproval
                  }
                ]"
              >
                <BalIcon
                  v-if="!requiresLidoRelayerApproval"
                  name="check"
                  class="text-green-500"
                />
                <template v-else>1</template>
              </div>
            </template>
            <div>
              <div class="mb-2 font-semibold">
                <div>
                  {{
                    $t(
                      'tradeSummary.transactionTypesTooltips.lidoRelayerApproval.title'
                    )
                  }}
                </div>
              </div>
              <div>
                {{
                  $t(
                    'tradeSummary.transactionTypesTooltips.lidoRelayerApproval.content'
                  )
                }}
              </div>
            </div>
          </BalTooltip>
          <div class="step-seperator" />
        </template>
        <template v-if="showTokenApprovalStep">
          <BalTooltip
            v-if="showTokenApprovalStep"
            :disabled="!requiresTokenApproval"
            width="64"
          >
            <template v-slot:activator>
              <div
                :class="[
                  'step',
                  {
                    'step-active': activeTransactionType === 'tokenApproval',
                    'step-approved': !requiresTokenApproval
                  }
                ]"
              >
                <BalIcon
                  v-if="!requiresTokenApproval"
                  name="check"
                  class="text-green-500"
                />
                <template v-else>{{
                  showGnosisRelayerApprovalStep || showLidoRelayerApprovalStep
                    ? 2
                    : 1
                }}</template>
              </div>
            </template>
            <div>
              <div class="mb-2 font-semibold">
                {{
                  $t(
                    'tradeSummary.transactionTypesTooltips.tokenApproval.title',
                    [trading.tokenIn.value.symbol]
                  )
                }}
              </div>
              <div>
                {{
                  $t(
                    'tradeSummary.transactionTypesTooltips.tokenApproval.content'
                  )
                }}
              </div>
            </div>
          </BalTooltip>
          <div class="step-seperator" />
        </template>
        <BalTooltip width="64">
          <template v-slot:activator>
            <div
              :class="[
                'step',
                {
                  'step-active': activeTransactionType === 'trade'
                }
              ]"
            >
              {{ totalRequiredTransactions }}
            </div>
          </template>
          <div>
            <div class="mb-2 font-semibold">
              {{
                trading.isGnosisTrade.value
                  ? $t('tradeSummary.transactionTypesTooltips.sign.title')
                  : $t('tradeSummary.transactionTypesTooltips.trade.title')
              }}
            </div>
            <div>
              {{
                trading.isGnosisTrade.value
                  ? $t('tradeSummary.transactionTypesTooltips.sign.content')
                  : $t('tradeSummary.transactionTypesTooltips.trade.content')
              }}
            </div>
          </div>
        </BalTooltip>
      </div>
      <BalBtn
        v-if="requiresGnosisRelayerApproval"
        color="gradient"
        block
        @click.prevent="gnosisRelayerApproval.approve"
        :loading="
          gnosisRelayerApproval.init.value ||
            gnosisRelayerApproval.approving.value
        "
        :loading-label="`${$t('approvingGnosisRelayer')}...`"
      >
        {{ $t('approveGnosisRelayer') }}
      </BalBtn>
      <BalBtn
        v-else-if="requiresLidoRelayerApproval"
        color="gradient"
        block
        @click.prevent="lidoRelayerApproval.approve"
        :loading="
          lidoRelayerApproval.init.value || lidoRelayerApproval.approving.value
        "
        :loading-label="`${$t('approvingLidoRelayer')}...`"
      >
        {{ $t('approveLidoRelayer') }}
      </BalBtn>
      <BalBtn
        v-else-if="requiresTokenApproval"
        :loading="tokenApproval.approving.value"
        :loading-label="`${$t('approving')} ${trading.tokenIn.value.symbol}...`"
        color="gradient"
        block
        @click.prevent="approveToken"
      >
        {{ `${$t('approve')} ${trading.tokenIn.value.symbol}` }}
      </BalBtn>
      <BalBtn
        v-else
        color="gradient"
        block
        @click.prevent="trade"
        :loading="trading.isConfirming.value"
        :loading-label="$t('confirming')"
        :disabled="tradeDisabled"
        class="relative"
      >
        {{ labels.confirmTrade }}
      </BalBtn>
      <BalAlert
        v-if="trading.submissionError.value != null"
        class="p-3 mt-4"
        type="error"
        size="md"
        :title="$t('tradeSubmissionError.title')"
        :description="trading.submissionError.value"
        block
        :action-label="$t('tradeSubmissionError.actionLabel')"
        @actionClick="trading.resetSubmissionError"
      />
    </div>
    <TradeRoute
      v-if="showTradeRoute"
      :address-in="trading.tokenIn.value.address"
      :amount-in="trading.tokenInAmountInput.value"
      :address-out="trading.tokenOut.value.address"
      :amount-out="trading.tokenOutAmountInput.value"
      :pools="trading.sor.pools.value"
      :sor-return="trading.sor.sorReturn.value"
      class="mt-3"
    />
  </BalModal>
</template>

<script lang="ts">
import { formatUnits } from '@ethersproject/units';
import { mapValues } from 'lodash';
import { computed, defineComponent, PropType, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';

import TradeRoute from '@/components/cards/TradeCard/TradeRoute.vue';
import { TradeQuote } from '@/composables/trade/types';
import useRelayerApproval, {
  Relayer
} from '@/composables/trade/useRelayerApproval';
import useTokenApproval from '@/composables/trade/useTokenApproval';
import { UseTrading } from '@/composables/trade/useTrading';
import useNumbers, { FNumFormats } from '@/composables/useNumbers';
import useTokens from '@/composables/useTokens';
import useUserSettings from '@/composables/useUserSettings';
import { FiatCurrency } from '@/constants/currency';
import { bnum } from '@/lib/utils';
import { isStETH } from '@/lib/utils/balancer/lido';
import { getWrapAction, WrapType } from '@/lib/utils/balancer/wrapper';
import useWeb3 from '@/services/web3/useWeb3';

const PRICE_UPDATE_THRESHOLD = 0.02;

export default defineComponent({
  components: {
    TradeRoute
  },
  emits: ['trade', 'close'],
  props: {
    trading: {
      type: Object as PropType<UseTrading>,
      required: true
    }
  },
  setup(props, { emit }) {
    // COMPOSABLES
    const { t } = useI18n();
    const { fNum2, toFiat } = useNumbers();
    const { tokens, approvalRequired } = useTokens();
    const { blockNumber } = useWeb3();
    const { slippage } = useUserSettings();

    // state
    const lastQuote = ref<TradeQuote | null>(
      props.trading.isWrapUnwrapTrade.value ? null : props.trading.getQuote()
    );
    const priceUpdated = ref(false);
    const priceUpdateAccepted = ref(false);

    // DATA
    const showSummaryInFiat = ref(false);

    // COMPUTED
    const slippageRatePercent = computed(() =>
      fNum2(slippage.value, FNumFormats.percent)
    );

    const addressIn = computed(() => props.trading.tokenIn.value.address);

    const tokenInFiatValue = computed(() =>
      fNum2(
        toFiat(
          props.trading.tokenInAmountInput.value,
          props.trading.tokenIn.value.address
        ),
        FNumFormats.fiat
      )
    );

    const tokenOutFiatValue = computed(() =>
      fNum2(
        toFiat(
          props.trading.tokenOutAmountInput.value,
          props.trading.tokenOut.value.address
        ),
        FNumFormats.fiat
      )
    );

    const showTradeRoute = computed(() => props.trading.isBalancerTrade.value);

    const zeroFee = computed(() =>
      showSummaryInFiat.value ? fNum2('0', FNumFormats.fiat) : '0.0 ETH'
    );

    const summary = computed(() => {
      const summaryItems = {
        amountBeforeFees: '',
        tradeFees: '',
        totalWithoutSlippage: '',
        totalWithSlippage: ''
      };

      const exactIn = props.trading.exactIn.value;

      const tokenIn = props.trading.tokenIn.value;
      const tokenOut = props.trading.tokenOut.value;

      const tokenInAmountInput = props.trading.tokenInAmountInput.value;
      const tokenOutAmountInput = props.trading.tokenOutAmountInput.value;

      if (props.trading.isWrapUnwrapTrade.value) {
        summaryItems.amountBeforeFees = tokenInAmountInput;
        summaryItems.tradeFees = '0';
        summaryItems.totalWithoutSlippage = tokenInAmountInput;
        summaryItems.totalWithSlippage = tokenInAmountInput;
      } else {
        const quote = props.trading.getQuote();

        if (exactIn) {
          summaryItems.amountBeforeFees = tokenOutAmountInput;
          summaryItems.tradeFees = formatUnits(
            quote.feeAmountOutToken,
            tokenOut.decimals
          );
          summaryItems.totalWithoutSlippage = bnum(
            summaryItems.amountBeforeFees
          )
            .minus(summaryItems.tradeFees)
            .toString();
          summaryItems.totalWithSlippage = formatUnits(
            quote.minimumOutAmount,
            tokenOut.decimals
          );
        } else {
          summaryItems.amountBeforeFees = tokenInAmountInput;
          summaryItems.tradeFees = formatUnits(
            quote.feeAmountInToken,
            tokenIn.decimals
          );
          summaryItems.totalWithoutSlippage = bnum(
            summaryItems.amountBeforeFees
          )
            .plus(summaryItems.tradeFees)
            .toString();
          summaryItems.totalWithSlippage = formatUnits(
            quote.maximumInAmount,
            tokenIn.decimals
          );
        }
      }

      if (showSummaryInFiat.value) {
        return mapValues(
          summaryItems,
          itemValue =>
            `${fNum2(
              toFiat(itemValue, exactIn ? tokenOut.address : tokenIn.address),
              FNumFormats.fiat
            )}`
        );
      } else {
        return mapValues(
          summaryItems,
          itemValue =>
            `${fNum2(itemValue, FNumFormats.token)} ${
              exactIn || props.trading.isWrapUnwrapTrade.value
                ? tokenOut.symbol
                : tokenIn.symbol
            }`
        );
      }
    });

    const labels = computed(() => {
      if (props.trading.isWrap.value) {
        return {
          modalTitle: t('previewWrap', [props.trading.tokenIn.value.symbol]),
          confirmTrade: t('confirmWrap', [props.trading.tokenIn.value.symbol]),
          tradeSummary: {
            title: t('tradeSummary.wrap.title'),
            tradeFees: t('tradeSummary.wrap.tradeFees'),
            totalBeforeFees: t('tradeSummary.wrap.totalBeforeFees'),
            totalAfterFees: t('tradeSummary.wrap.totalAfterFees'),
            totalWithSlippage: t('tradeSummary.wrap.totalWithSlippage', [
              props.trading.tokenIn.value.symbol
            ])
          }
        };
      } else if (props.trading.isUnwrap.value) {
        return {
          modalTitle: t('previewUnwrap', [props.trading.tokenOut.value.symbol]),
          confirmTrade: t('confirmUnwrap', [
            props.trading.tokenOut.value.symbol
          ]),
          tradeSummary: {
            title: t('tradeSummary.unwrap.title'),
            tradeFees: t('tradeSummary.unwrap.tradeFees'),
            totalBeforeFees: t('tradeSummary.unwrap.totalBeforeFees'),
            totalAfterFees: t('tradeSummary.unwrap.totalAfterFees'),
            totalWithSlippage: t('tradeSummary.unwrap.totalWithSlippage', [
              props.trading.tokenOut.value.symbol
            ])
          }
        };
      } else if (props.trading.exactIn.value) {
        return {
          modalTitle: t('previewTrade'),
          confirmTrade: t('confirmTrade'),
          tradeSummary: {
            title: t('tradeSummary.exactIn.title', [
              props.trading.tokenIn.value.symbol
            ]),
            tradeFees: t('tradeSummary.exactIn.tradeFees'),
            totalBeforeFees: t('tradeSummary.exactIn.totalBeforeFees'),
            totalAfterFees: t('tradeSummary.exactIn.totalAfterFees'),
            totalWithSlippage: t('tradeSummary.exactIn.totalWithSlippage', [
              slippageRatePercent.value
            ])
          }
        };
      }
      // exact out
      return {
        modalTitle: t('previewTrade'),
        confirmTrade: t('confirmTrade'),
        tradeSummary: {
          title: t('tradeSummary.exactOut.title', [
            props.trading.tokenOut.value.symbol
          ]),
          tradeFees: t('tradeSummary.exactOut.tradeFees'),
          totalBeforeFees: t('tradeSummary.exactOut.totalBeforeFees'),
          totalAfterFees: t('tradeSummary.exactOut.totalAfterFees'),
          totalWithSlippage: t('tradeSummary.exactOut.totalWithSlippage', [
            slippageRatePercent.value
          ])
        }
      };
    });

    const tokenApproval = useTokenApproval(
      addressIn,
      props.trading.tokenInAmountInput,
      tokens
    );

    const gnosisRelayerApproval = useRelayerApproval(
      Relayer.GNOSIS,
      props.trading.isGnosisTrade
    );

    const wrapType = computed(() =>
      getWrapAction(
        props.trading.tokenIn.value.address,
        props.trading.tokenOut.value.address
      )
    );

    const isStETHTrade = computed(
      () =>
        isStETH(addressIn.value, props.trading.tokenOut.value.address) &&
        wrapType.value === WrapType.NonWrap
    );

    const lidoRelayerApproval = useRelayerApproval(Relayer.LIDO, isStETHTrade);

    const requiresTokenApproval = computed(() => {
      if (props.trading.isWrap.value && !props.trading.isEthTrade.value) {
        return approvalRequired(
          props.trading.tokenIn.value.address,
          props.trading.tokenInAmountInput.value,
          props.trading.tokenOut.value.address
        );
      } else if (props.trading.requiresTokenApproval.value) {
        return !tokenApproval.isUnlockedV2.value;
      }
      return false;
    });

    const requiresGnosisRelayerApproval = computed(
      () =>
        props.trading.isGnosisTrade.value &&
        props.trading.requiresTokenApproval.value &&
        !gnosisRelayerApproval.isUnlocked.value
    );

    const requiresLidoRelayerApproval = computed(
      () =>
        props.trading.isBalancerTrade.value &&
        !lidoRelayerApproval.isUnlocked.value
    );

    const showTokenApprovalStep = computed(
      () =>
        requiresTokenApproval.value ||
        tokenApproval.approved.value ||
        tokenApproval.approving.value
    );

    const showGnosisRelayerApprovalStep = computed(
      () =>
        requiresGnosisRelayerApproval.value ||
        gnosisRelayerApproval.init.value ||
        gnosisRelayerApproval.approved.value ||
        gnosisRelayerApproval.approving.value
    );

    const showLidoRelayerApprovalStep = computed(
      () =>
        requiresLidoRelayerApproval.value ||
        lidoRelayerApproval.init.value ||
        lidoRelayerApproval.approved.value ||
        lidoRelayerApproval.approving.value
    );

    const totalRequiredTransactions = computed(() => {
      let txCount = 1; // trade
      if (showTokenApprovalStep.value) {
        txCount++;
      }

      if (showGnosisRelayerApprovalStep.value) {
        txCount++;
      } else if (showLidoRelayerApprovalStep.value) {
        txCount++;
      }

      return txCount;
    });

    const activeTransactionType = computed<
      | 'gnosisRelayerApproval'
      | 'lidoRelayerApproval'
      | 'tokenApproval'
      | 'trade'
    >(() => {
      if (requiresGnosisRelayerApproval.value) {
        return 'gnosisRelayerApproval';
      }
      if (requiresLidoRelayerApproval.value) {
        return 'lidoRelayerApproval';
      }
      if (requiresTokenApproval.value) {
        return 'tokenApproval';
      }
      return 'trade';
    });

    const requiresApproval = computed(
      () =>
        requiresGnosisRelayerApproval.value ||
        requiresLidoRelayerApproval.value ||
        requiresTokenApproval.value
    );

    const showPriceUpdateError = computed(
      () =>
        !requiresApproval.value &&
        priceUpdated.value &&
        !priceUpdateAccepted.value
    );

    const tradeDisabled = computed(
      () => requiresApproval.value || showPriceUpdateError.value
    );

    // METHODS
    function trade() {
      emit('trade');
    }

    function onClose() {
      emit('close');
    }

    function cofirmPriceUpdate() {
      priceUpdated.value = false;
      priceUpdateAccepted.value = true;
      lastQuote.value = props.trading.getQuote();
    }

    function handlePriceUpdate() {
      if (lastQuote.value != null) {
        const newQuote = props.trading.getQuote();

        /**
         * The bignumber returned via the quotes for some reason throw underflow
         * errors when attempting to use the gt function with the threshold value.
         * For that reason, the price difference has to be cast to our bignumber type.
         */
        if (props.trading.exactIn.value) {
          const priceDiff = lastQuote.value.minimumOutAmount
            .sub(newQuote.minimumOutAmount)
            .abs()
            .div(lastQuote.value.minimumOutAmount);

          priceUpdated.value = bnum(priceDiff.toString()).gt(
            PRICE_UPDATE_THRESHOLD
          );
        } else {
          const priceDiff = lastQuote.value.maximumInAmount
            .sub(newQuote.maximumInAmount)
            .abs()
            .div(lastQuote.value.maximumInAmount);

          priceUpdated.value = bnum(priceDiff.toString()).gt(
            PRICE_UPDATE_THRESHOLD
          );
        }

        if (priceUpdated.value) {
          priceUpdateAccepted.value = false;
        }
      }
    }

    async function approveToken(): Promise<void> {
      if (props.trading.isWrap.value && !props.trading.isEthTrade.value) {
        // If we're wrapping a token other than native ETH
        // we need to approve the underlying on the wrapper
        await tokenApproval.approveSpender(
          props.trading.tokenOut.value.address
        );
      } else {
        await tokenApproval.approveV2();
      }
    }

    // WATCHERS
    watch(blockNumber, () => {
      handlePriceUpdate();
    });

    return {
      // constants
      FiatCurrency,
      FNumFormats,

      // methods
      fNum2,
      onClose,
      trade,
      cofirmPriceUpdate,

      // computed
      tokenInFiatValue,
      tokenOutFiatValue,
      summary,
      showSummaryInFiat,
      slippageRatePercent,
      showTradeRoute,
      labels,
      zeroFee,
      priceUpdated,
      tradeDisabled,
      showPriceUpdateError,
      totalRequiredTransactions,
      requiresApproval,
      gnosisRelayerApproval,
      lidoRelayerApproval,
      tokenApproval,
      requiresTokenApproval,
      requiresGnosisRelayerApproval,
      requiresLidoRelayerApproval,
      approveToken,
      showTokenApprovalStep,
      showGnosisRelayerApprovalStep,
      showLidoRelayerApprovalStep,
      activeTransactionType,

      // consants
      PRICE_UPDATE_THRESHOLD
    };
  }
});
</script>
<style scoped>
.arrow-down {
  @apply absolute right-0 rounded-full border border-gray-100 flex items-center h-8 w-8 justify-center bg-white mr-3 dark:border-gray-800 dark:bg-gray-800;
  transform: translateY(-50%);
}

.summary-item-row {
  @apply flex justify-between mb-1;
}

.step {
  @apply rounded-full w-7 h-7 border border-gray-100 dark:border-gray-700 flex items-center justify-center text-purple-500 relative;
}
.step-seperator {
  @apply bg-gray-200 dark:bg-gray-700 h-px w-6;
}

.step-active {
  @apply border-purple-500 dark:border-purple-500;
}
.step-approved {
  @apply border-green-500 dark:border-green-500;
}
</style>
