<script setup lang="ts">
import { OrderStatusEnum } from '#woo';

const { query, params, name } = useRoute();
const { customer } = useAuth();
const { formatDate, formatPrice } = useHelpers();
const { t } = useI18n();

const config = useRuntimeConfig()
const order = ref<Order>({});
const trackingInfo = ref([]);
const fetchDelay = ref<boolean>(query.fetch_delay === 'true');
const delayLength = 2500;
const isLoaded = ref<boolean>(false);
const errorMessage = ref('');

const isGuest = computed(() => !customer.value?.databaseId);
const isSummaryPage = computed<boolean>(() => name === 'order-summary');
const isCheckoutPage = computed<boolean>(() => name === 'order-received');
const orderIsNotCompleted = computed<boolean>(() => order.value?.status !== OrderStatusEnum.COMPLETED);
const hasDiscount = computed<boolean>(() => !!parseFloat(order.value?.rawDiscountTotal || '0'));
const downloadableItems = computed(() => order.value?.downloadableItems?.nodes || []);

onBeforeMount(() => {
  if (isCheckoutPage.value && (query.cancel_order || query.from_paypal || query.PayerID)) window.close();
});

onMounted(async () => {
  await getOrder();
  await getTrackingInfo();
  if (isCheckoutPage.value && fetchDelay.value && orderIsNotCompleted.value) {
    setTimeout(() => {
      getOrder();
    }, delayLength);
  }
});

async function getOrder() {
  try {
    const data = await GqlGetOrder({ id: params.orderId as string });
    if (data.order) order.value = data.order;
  } catch (err: any) {
    errorMessage.value = err?.gqlErrors?.[0].message || 'Could not find order';
  }
  isLoaded.value = true;
}

async function getTrackingInfo() {
  try {
    const backendUrl = config.public.apiUrl; // Use the environment variable

    const response = await fetch(`${backendUrl}/wp-json/trackship/v1/get_tracking_info`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        order_id: params.orderId,
        order_email: customer.value.email,
      }),
    });
    const data = await response.json();
    if (data.success === 'true') {
      trackingInfo.value = data.tracking_data;
    } else {
      errorMessage.value = data.message || 'Could not fetch tracking information';
    }
  } catch (err) {
    errorMessage.value = 'Error fetching tracking information';
  }
}

const refreshOrder = async () => {
  isLoaded.value = false;
  await getOrder();
  await getTrackingInfo();
};

useSeoMeta({
  title() {
    return isSummaryPage.value ? t('messages.shop.orderSummary') : t('messages.shop.orderReceived');
  },
});
</script>

<template>
  <div class="w-full min-h-[600px] flex items-center p-8 text-gray-800 md:bg-white md:rounded-xl md:mx-auto md:shadow-lg md:my-24 md:mt-8 md:max-w-3xl md:p-16 flex-col">
    <LoadingIcon v-if="!isLoaded" class="flex-1" />
    <template v-else>
      <div class="w-full">
        <template v-if="isSummaryPage">
          <div class="flex items-center gap-4">
            <NuxtLink to="/my-account?tab=orders" class="inline-flex items-center justify-center p-2 border rounded-md" title="Back to orders" aria-label="Back to orders">
              <Icon name="ion:chevron-back-outline" />
            </NuxtLink>
            <h1 class="text-xl font-semibold">{{ $t('messages.shop.orderSummary') }}</h1>
          </div>
        </template>
        <template v-else-if="isCheckoutPage">
          <div class="flex items-center justify-between w-full mb-2">
            <h1 class="text-xl font-semibold">{{ $t('messages.shop.orderReceived') }}</h1>
            <button v-if="orderIsNotCompleted" type="button" class="inline-flex items-center justify-center p-2 bg-white border rounded-md" title="Refresh order" aria-label="Refresh order" @click="refreshOrder">
              <Icon name="ion:refresh-outline" />
            </button>
          </div>
          <p>{{ $t('messages.shop.orderThanks') }}</p>
        </template>
        <hr class="my-8" />
      </div>
      <div v-if="order && !isGuest" class="flex-1 w-full">
        <!-- Existing order details code -->

        <!-- Tracking information section -->
        <div v-if="trackingInfo.length">
          <h2 class="text-lg font-semibold">Tracking Information</h2>
          <div v-for="info in trackingInfo" :key="info.tracking_number" class="tracking-info">
            <div>
              <span class="font-semibold">Carrier:</span> {{ info.carrier }}
            </div>
            <div>
              <span class="font-semibold">Tracking Number:</span> {{ info.tracking_number }}
            </div>
            <div>
              <span class="font-semibold">Date Shipped:</span> {{ formatDate(info.date_shipped) }}
            </div>
            <div>
              <span class="font-semibold">Status:</span> {{ info.status }}
            </div>
            <div v-if="info.est_delivery_date">
              <span class="font-semibold">Estimated Delivery Date:</span> {{ formatDate(info.est_delivery_date) }}
            </div>
            <div v-if="info.events.length">
              <h3 class="font-semibold">Tracking Events:</h3>
              <ul>
                <li v-for="event in info.events" :key="event.date">{{ event.date }}: {{ event.status }}</li>
              </ul>
            </div>
          </div>
          <hr class="my-8" />
        </div>

        <!-- Existing order details code continues -->
      </div>
      <div v-else-if="errorMessage" class="flex flex-col items-center justify-center flex-1 w-full gap-4 text-center">
        <Icon name="ion:sad-outline" size="96" class="text-gray-700" />
        <h1 class="text-xl font-semibold">Error</h1>
        <div v-if="errorMessage" class="text-sm text-red-500" v-html="errorMessage" />
      </div>
    </template>
  </div>
</template>
