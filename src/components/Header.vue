<script setup>
import { computed, inject, onMounted, reactive, ref, watch } from "vue";
import Tabs from "primevue/tabs";
import TabList from "primevue/tablist";
import Tab from "primevue/tab";
import { useToast } from "primevue/usetoast";
import Toast from "primevue/toast";
import Button from "primevue/button";
const { t } = useI18n();
import { useI18n } from "vue-i18n";
import { toRaw } from "vue";
import TreeTable from "primevue/treetable";
import Tag from "primevue/tag";
import DataTable from "primevue/datatable";
import Column from "primevue/column";
import ColumnGroup from "primevue/columngroup"; // optional
import Row from "primevue/row"; // optional

import Dialog from "primevue/dialog";
const { locale } = useI18n();
const toast = useToast();
const language = ref(localStorage.getItem("language"));

const setLocale = () => {
  if (language.value === "vi") {
    locale.value = "en";
    language.value = "en";
    localStorage.setItem("language", "en");
  } else {
    locale.value = "vi";
    language.value = "vi";
    localStorage.setItem("language", "vi");
  }
};
const loading = ref(false);
import { useRouter } from "vue-router";
import Card from "./Card.vue";
const nodes = ref([]);
const visible = ref(false);
const showOrder = async () => {
  visible.value = true;
};
const router = useRouter();

import axios from "axios";
import { formatVNDate } from "@/helper/date";

const API_BASE_URL = "http://localhost:5097/api";

const categories = reactive([]);
const products = reactive([]);
onMounted(async () => {
  loading.value = true;
  try {
    if (!localStorage.getItem("language")) {
      localStorage.setItem("language", locale.value);
    }
    if (!sessionStorage.getItem("carts")) {
      sessionStorage.setItem("carts", JSON.stringify([]));
    }
    locale.value = localStorage.getItem("language");
    if (sessionStorage.getItem("order")) {
      toast.add({
        severity: "success",
        summary: t("message.orderSuccess"),
        detail: t("message.contentOrderSuccess"),
        life: 3000,
      });

      sessionStorage.removeItem("order");
    }
    const getCategories = async () => {
      try {
        const response = await axios.get(`${API_BASE_URL}/categories`, {
          headers: {
            Accept: "application/json",
            "Content-Type": "application/json",
            "ngrok-skip-browser-warning": "69420",
          },
        });

        if (response.data.StatusCode === 200) {
          return response.data;
        }
      } catch (error) {
        console.error("Error fetching categories:", error);
        throw error;
      }
    };
    const getProducts = async () => {
      try {
        const response = await axios.get(`${API_BASE_URL}/products`, {
          headers: {
            Accept: "application/json",
            "Content-Type": "application/json",
            "ngrok-skip-browser-warning": "69420",
          },
        });

        if (response.data.StatusCode === 200) {
          return response.data;
        }
      } catch (error) {
        console.error("Error fetching categories:", error);
        throw error;
      }
    };
    const getOrders = async () => {
      try {
        const response = await axios.get(`${API_BASE_URL}/orderProduct`, {
          headers: {
            Accept: "application/json",
            "Content-Type": "application/json",
            "ngrok-skip-browser-warning": "69420",
          },
        });

        if (response.data.StatusCode === 200) {
          return response.data;
        }
      } catch (error) {
        console.error("Error fetching categories:", error);
        throw error;
      }
    };
    const categoriesValue = await getCategories();

    categories.value = categoriesValue.Data;

    const productsValue = await getProducts();

    products.value = productsValue.Data;

    const orders = await getOrders();

    orders.Data.forEach((order, index) => {
      let total = 0;
      nodes.value.push({
        key: (index + 1).toString(),
        data: {
          orderId: order.OrderId,
          statusName: order.StatusName,
          createdTime: formatVNDate(order.CreatedTime),
        },
        key: (index + 1).toString(),
        data: {
          orderId: order.OrderId,
          statusName: order.StatusName,
          createdTime: formatVNDate(order.CreatedTime),
        },

        children: [],
      });
      nodes.value[index].children.push({
        // key: `${index + 1}-${i + 1}`,
        data: {
          orderId: "Tên sản phẩm",
          statusName: "Giá tiền",
          createdTime: "Tổng",
        },
        style: { fontWeight: "bold" },
      });
      let jsonStr = `[${order.Product}]`;

      let data = JSON.parse(jsonStr);

      data.forEach((product, i) => {
        total += product.totalPrice;
        nodes.value[index].children.push({
          key: `${index + 1}-${i + 1}`,
          data: {
            orderId:
              (language.value === "vi" ? product.nameVi : product.nameEn) +
              " x" +
              product.count,
            statusName: product.price.toLocaleString("vi-VN") + "đ" + " / 1",
            createdTime: product.totalPrice.toLocaleString("vi-VN") + "đ",
          },
        });
      });

      nodes.value[index].children.push({
        // key: `${index + 1}-${i + 1}`,
        data: {
          orderId: "Tổng tiền",

          createdTime: total.toLocaleString("vi-VN") + "đ",
        },
        style: { fontWeight: "bold" },
      });
    });
  } catch (error) {
    console.error(error);
  } finally {
    loading.value = false; // kết thúc loading
  }
});

const category = ref("All");

const filter = (category) => {
  if (category === "All") {
    return products.value?.filter((product) => {
      return categories.value?.filter((category) => {
        return product.CategoryId.includes(category.Id);
      });
    });
  }
  return products.value?.filter((product) => {
    return product.CategoryId.some((x) => x === category);
  });
};

const count = ref(0);

const data = sessionStorage.getItem("carts");
const carts = reactive(JSON.parse(data) ?? []);
carts.forEach((cart) => {
  count.value += cart.count;
});
function handlePrice(quantity) {
  count.value = quantity;
}

function goToOrderInfo() {
  router.push("order-info");
}

const expandedRows = ref({});

const onRowExpand = (event) => {
  toast.add({
    severity: "info",
    summary: "Product Expanded",
    detail: event.data.name,
    life: 3000,
  });
};
const onRowCollapse = (event) => {
  toast.add({
    severity: "success",
    summary: "Product Collapsed",
    detail: event.data.name,
    life: 3000,
  });
};
const expandAll = () => {
  expandedRows.value = products.value.reduce((acc, p) => (acc[p.id] = true) && acc, {});
};
const collapseAll = () => {
  expandedRows.value = null;
};
const formatCurrency = (value) => {
  return value.toLocaleString("en-US", { style: "currency", currency: "USD" });
};
const getSeverity = (product) => {
  switch (product.inventoryStatus) {
    case "INSTOCK":
      return "success";

    case "LOWSTOCK":
      return "warn";

    case "OUTOFSTOCK":
      return "danger";

    default:
      return null;
  }
};
const getOrderSeverity = (order) => {
  switch (order.status) {
    case "DELIVERED":
      return "success";

    case "CANCELLED":
      return "danger";

    case "PENDING":
      return "warn";

    case "RETURNED":
      return "info";

    default:
      return null;
  }
};
</script>

<template>
  <Toast />
  <Dialog v-model:visible="visible" modal :style="{ width: '60vw' }">
    <div class="card">
      <DataTable
        v-model:expandedRows="expandedRows"
        :value="orders"
        dataKey="id"
        @rowExpand="onRowExpand"
        @rowCollapse="onRowCollapse"
        tableStyle="min-width: 60rem"
      >
        <template #header>
          <div class="flex flex-wrap justify-end gap-2">
            <Button
              variant="text"
              icon="pi pi-plus"
              label="Expand All"
              @click="expandAll"
            />
            <Button
              variant="text"
              icon="pi pi-minus"
              label="Collapse All"
              @click="collapseAll"
            />
          </div>
        </template>
        <Column expander style="width: 5rem" />
        <Column field="name" header="Name"></Column>
        <Column header="Image">
          <template #body="slotProps">
            <img
              :src="`https://primefaces.org/cdn/primevue/images/product/${slotProps.data.image}`"
              :alt="slotProps.data.image"
              class="shadow-lg"
              width="64"
            />
          </template>
        </Column>
        <Column field="price" header="Price">
          <template #body="slotProps">
            {{ formatCurrency(slotProps.data.price) }}
          </template>
        </Column>
        <Column field="category" header="Category"></Column>

        <Column header="Status">
          <template #body="slotProps">
            <Tag
              :value="slotProps.data.inventoryStatus"
              :severity="getSeverity(slotProps.data)"
            />
          </template>
        </Column>
        <template #expansion="slotProps">
          <div class="p-4">
            <h5>Orders for {{ slotProps.data.name }}</h5>
            <DataTable :value="slotProps.data.orders">
              <Column field="id" header="Id" sortable></Column>
              <Column field="customer" header="Customer" sortable></Column>
              <Column field="date" header="Date" sortable></Column>
              <Column field="amount" header="Amount" sortable>
                <template #body="slotProps">
                  {{ formatCurrency(slotProps.data.amount) }}
                </template>
              </Column>
              <Column field="status" header="Status" sortable>
                <template #body="slotProps">
                  <Tag
                    :value="slotProps.data.status.toLowerCase()"
                    :severity="getOrderSeverity(slotProps.data)"
                  />
                </template>
              </Column>
              <Column headerStyle="width:4rem">
                <template #body>
                  <Button icon="pi pi-search" />
                </template>
              </Column>
            </DataTable>
          </div>
        </template>
      </DataTable>
      <Toast />
    </div>
  </Dialog>
  <div class="p-2">
    <label class="text-2xl font-bold text-blue-600">{{ $t("message.title") }}</label>
  </div>
  <div>
    <label class="text-sm text-600">{{ $t("message.subTitle") }}</label>
  </div>
  <div class="flex justify-content-between">
    <Tabs v-model:value="category">
      <TabList>
        <Tab key="All" value="All">{{ language === "vi" ? "Tất cả" : "All" }}</Tab>
        <Tab
          v-for="category in categories.value"
          :key="category.Id"
          :value="category.Id"
          >{{ language === "vi" ? category.NameVi : category.NameEn }}</Tab
        >
      </TabList>
    </Tabs>

    <div class="flex">
      <Button
        class="m-2"
        icon="pi pi-receipt"
        severity="info"
        rounded
        @click="showOrder()"
      />
      <Button
        class="m-2"
        icon="pi pi-globe"
        :label="language === 'vi' ? 'EN' : 'VI'"
        @click="setLocale()"
        severity="info"
        rounded
      />
      <div class="relative w-fit">
        <Button
          @click="goToOrderInfo"
          class="m-2"
          icon="pi pi-shopping-cart"
          :disabled="!count"
          severity="info"
          rounded
        />

        <span
          class="absolute -top-1 -right-1 bg-blue-600 text-white text-xs font-semibold rounded-full w-5 h-5 flex items-center justify-center"
          v-if="count"
        >
          {{ count }}
        </span>
      </div>
    </div>
  </div>
  <div>
    <!-- Loading spinner -->
    <div v-if="loading" class="flex justify-center items-center h-64">
      <i class="pi pi-spin pi-spinner text-3xl"></i>
    </div>

    <!-- Dữ liệu đã load xong -->
    <div v-else>
      <Card @data="handlePrice" :products="filter(category)" :language="language" />
    </div>
  </div>
</template>

<style></style>
