<script setup lang="ts">
import { onBeforeMount, ref, toRefs } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRouter } from 'vue-router';

import TradeSettingsPopover, {
  TradeSettingsContext
} from '@/components/popovers/TradeSettingsPopover.vue';
import { MIN_FIAT_VALUE_POOL_MIGRATION } from '@/constants/pools';
import { bnum } from '@/lib/utils';
import { configService } from '@/services/config/config.service';
import { Pool } from '@/services/pool/types';
import { TokenInfo } from '@/types/TokenList';

import useMigrateMath from '../../composables/useMigrateMath';
import { PoolMigrationInfo } from '../../types';
import MigratePreviewModal from '../MigratePreviewModal/MigratePreviewModal.vue';
import PoolInfoBreakdown from './components/PoolInfoBreakdown.vue';

type Props = {
  poolMigrationInfo: PoolMigrationInfo;
  fromPool: Pool;
  toPool: Pool;
  fromPoolTokenInfo: TokenInfo;
  toPoolTokenInfo: TokenInfo;
};

/**
 * PROPS
 */
const props = defineProps<Props>();

/**
 * STATE
 */
const showPreviewModal = ref(false);

/**
 * COMPOSABLES
 */
const { t } = useI18n();

const { fromPool, toPool } = toRefs(props);

const router = useRouter();

const migrateMath = useMigrateMath(fromPool, toPool);
const { hasBpt, fiatTotalLabel, fiatTotal } = migrateMath;

/**
 * CALLBACKS
 */
onBeforeMount(() => {
  migrateMath.getBatchSwap();

  if (bnum(fiatTotal.value).lt(MIN_FIAT_VALUE_POOL_MIGRATION)) {
    router.push({ name: 'pool', params: { id: fromPool.value.id } });
  }
});
</script>

<template>
  <BalCard shadow="xl" exposeOverflow noBorder>
    <template #header>
      <div class="w-full">
        <div class="text-xs text-secondary leading-none">
          {{ configService.network.chainName }}
        </div>
        <div class="flex items-center justify-between">
          <h4>
            {{ t(`migratePool.${poolMigrationInfo.type}.migrateToPool.title`) }}
          </h4>
          <TradeSettingsPopover :context="TradeSettingsContext.invest" />
        </div>
      </div>
    </template>
    <div class="mb-6">
      <div class="text-secondary">{{ $t('yourBalanceInPool') }}</div>
      <div class="font-semibold text-lg">
        {{ hasBpt ? fiatTotalLabel : '-' }}
      </div>
    </div>
    <PoolInfoBreakdown :pool="fromPool" :poolTokenInfo="fromPoolTokenInfo" />
    <div class="block flex justify-center dark:text-gray-50 my-4">
      <ArrowDownIcon class="dark:text-secondary h-5 w-5" />
    </div>
    <PoolInfoBreakdown :pool="toPool" :poolTokenInfo="toPoolTokenInfo" />
    <BalBtn
      color="gradient"
      class="mt-6"
      block
      :disabled="!hasBpt"
      @click="showPreviewModal = true"
    >
      {{ $t('previewMigrate') }}
    </BalBtn>
  </BalCard>

  <teleport to="#modal">
    <MigratePreviewModal
      v-if="showPreviewModal"
      :fromPool="fromPool"
      :toPool="toPool"
      :fromPoolTokenInfo="fromPoolTokenInfo"
      :toPoolTokenInfo="toPoolTokenInfo"
      :poolMigrationInfo="poolMigrationInfo"
      :math="migrateMath"
      @close="showPreviewModal = false"
    />
  </teleport>
</template>
