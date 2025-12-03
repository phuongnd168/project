<script setup>
import Card from "primevue/card";
import { computed, onMounted, reactive, ref, watch } from "vue";
import { Form } from "vee-validate";
import { useForm } from "vee-validate";
import * as yup from "yup";
import { useI18n } from "vue-i18n";
const { t } = useI18n();
const { locale } = useI18n();

import Dialog from "primevue/dialog";
import InputText from "primevue/inputtext";
import RadioButton from "primevue/radiobutton";
import Button from "primevue/button";
import axios from "axios";
import router from "@/router";
const loading = ref(false);
const getBill = ref("noReceipt");
const showBill = () => {
  if (getBill.value === "noReceipt") {
    return false;
  }
  return true;
};
onMounted(() => {
  try {
    if (!localStorage.getItem("language")) {
      localStorage.setItem("language", locale.value);
    }
    if (!sessionStorage.getItem("carts")) {
      sessionStorage.setItem("carts", JSON.stringify([]));
    }

    locale.value = localStorage.getItem("language");
  } catch (error) {
    console.error(error);
  }
});
const data = sessionStorage.getItem("carts");

const carts = reactive(JSON.parse(data) ?? []);
const language = ref(localStorage.getItem("language"));
const check = computed(() => {
  return carts?.length;
});
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
const addCart = (product) => {
  const index = carts.findIndex((cart) => cart.id === product.id);

  if (index >= 0) {
    carts[index].count += 1;
  }
};

const removeCart = (product) => {
  const index = carts.findIndex((cart) => cart.id === product.id);
  if (index >= 0) {
    carts[index].count -= 1;
  }
};
const deleteCart = (product) => {
  const index = carts.findIndex((cart) => cart.id === product.id);
  if (index >= 0) {
    carts.splice(index, 1);
  }
};
const checkDisable = (product) => {
  const index = carts.findIndex((cart) => cart.id === product.id);
  if (index >= 0 && carts[index].count === 1) {
    return true;
  }
  return false;
};
watch(
  carts,
  () => {
    sessionStorage.setItem("carts", JSON.stringify(carts));
  },
  { deep: true }
);
const price = computed(() => {
  let value = 0;
  carts?.forEach((cart) => {
    value += cart.price * cart.count;
  });
  return value;
});
const count = (product) => {
  let count = null;
  carts.find((cart) => {
    if (cart.id === product.id) {
      count = cart.count;
    }
  });
  return count;
};
const addString = () => (carts?.length > 1 && language.value === "en" ? "s" : "");

const { defineField, errors, handleSubmit } = useForm({
  validationSchema: yup.object({
    name: yup.string().required(() => t("message.requiredNameTitle")),
    phone: yup
      .string()
      .matches(/(03|05|07|08|09|01[2|6|8|9])+([0-9]{8})\b/, {
        message: () => t("message.formatPhoneTitle"),
        excludeEmptyString: true,
      })
      .notRequired(),
    taxCode: yup.string().when({
      is: () => getBill.value === "getInvoice", // điều kiện phải validate
      then: (schema) =>
        schema
          .required(() => t("message.requiredTaxCode"))
          .matches(/^(?:\d{10}|\d{10}-\d{3}|\d{12})$/, () =>
            t("message.formatTaxCodeTitle")
          ),
      otherwise: (schema) => schema.notRequired(),
    }),
  }),
});
const [phone, phoneAttrs] = defineField("phone");
const [name, nameAttrs] = defineField("name");

const [taxCode, taxCodeAttrs] = defineField("taxCode");
const [budgetRelationshipCode, budgetRelationshipCodeAttrs] = defineField(
  "budgetRelationshipCode"
);

const [companyName, companyNameAttrs] = defineField("companyName");
const [companyAddress, companyAddressAttrs] = defineField("companyAddress");
const values = reactive({});
const onSubmit = handleSubmit((data) => {
  if (getBill.value === "noReceipt") {
    values.name = data.name;
    values.phone = data.phone;
  } else {
    values.name = data.name;
    values.phone = data.phone;
    values.taxCode = data.taxCode;

    values.budgetRelationshipCode = data.budgetRelationshipCode;
    values.companyName = data.companyName;
    values.companyAddress = data.companyAddress;
  }

  showConfirm();
});

const nameDialog = ref("");
const visible = ref(false);
const showConfirm = () => {
  visible.value = true;

  nameDialog.value =
    t("message.yourOrderTitle") +
    " (" +
    (check.value ?? 0) +
    " " +
    t("message.orderUnit") +
    addString() +
    ")";
};
const checkCount = (product) => {
  return count(product) >= product.quantity;
};
async function OrderComplete() {
  loading.value = true;
  try {
    const dataPatch = [];

    carts.forEach((cart) => {
      dataPatch.push({
        id: cart.id,
        count: cart.count,
      });
    });
    const dataPost = [];
    carts.forEach((cart) => {
      dataPost.push({
        id: cart.id,
        count: cart.count,
        price: cart.price,
        totalPrice: cart.price * cart.count,
        nameEn: cart.nameEn,
        nameVi: cart.nameVi,
        img: cart.img,
      });
    });

    const value = dataPost.map((x) => JSON.stringify(x)).join(",");

    const responsePost = await axios.post(`http://localhost:5097/api/orderProduct`, {
      headers: {
        Accept: "application/json",
        "Content-Type": "application/json",
      },

      userId: 8,
      products: value,
    });

    const responsePatch = await axios.patch(
      `http://localhost:5097/api/products`,

      {
        dataPatch,
        headers: {
          Accept: "application/json",
          "Content-Type": "application/json",
        },
      }
    );

    if (responsePost.data.StatusCode === 201 && responsePatch.data.StatusCode === 200) {
      sessionStorage.removeItem("carts");
      sessionStorage.setItem("order", true);
      router.push({ name: "Home" });
    }
  } catch (error) {
    console.error("Failed to update product:", error);
  } finally {
    loading.value = false;
  }
}
function goToHome() {
  router.push({ name: "Home" });
}
</script>
<template>
  <div class="flex justify-content-center">
    <Button
      class="m-2"
      icon="pi pi-arrow-left"
      @click="goToHome"
      severity="info"
      rounded
    />
    <div class="w-6">
      <h2>{{ $t("message.orderInfoTitle") }}</h2>
      <p>{{ $t("message.orderInfoSubTitle") }}</p>
    </div>

    <Button
      class="m-2"
      icon="pi pi-globe"
      severity="info"
      @click="setLocale()"
      :label="language === 'vi' ? 'EN' : 'VI'"
      rounded
    />
  </div>
  <div class="flex justify-content-center">
    <Card class="m-2 w-7">
      <template #title>{{
        $t("message.yourOrderTitle") +
        " (" +
        (check ?? 0) +
        " " +
        $t("message.orderUnit") +
        addString() +
        ")"
      }}</template>
      <template #content>
        <Dialog
          v-model:visible="visible"
          modal
          :style="{ width: '480px' }"
          class="rounded-3xl overflow-hidden"
        >
          <!-- HEADER -->
          <template #header>
            <div class="flex items-center justify-between w-full">
              <h2 class="text-xl font-bold text-gray-800">
                {{ nameDialog }}
              </h2>
            </div>
          </template>

          <!-- BODY -->
          <div class="p-5 space-y-6">
            <!-- Danh sách sản phẩm -->
            <div class="space-y-4">
              <div
                v-for="product in carts"
                :key="product.id"
                class="flex gap-3 bg-gray-50 p-3 rounded-xl shadow-sm border border-gray-200"
              >
                <img
                  :src="product.img"
                  class="w-16 h-16 object-cover rounded-lg shadow"
                  alt="product"
                />

                <div class="flex-1">
                  <div class="font-semibold text-gray-800">
                    {{ language === "vi" ? product.nameVi : product.nameEn }}
                    {{
                      "(" +
                      product.price.toLocaleString("vi-VN") +
                      "đ" +
                      " x " +
                      product.count +
                      ")"
                    }}
                  </div>

                  <div class="text-blue-600 font-bold mt-1">
                    {{ (product.price * product.count).toLocaleString("vi-VN") }}đ
                  </div>
                </div>
              </div>
            </div>

            <!-- Thông tin khách hàng -->
            <div class="bg-white rounded-2xl p-4 shadow border border-gray-100 space-y-2">
              <h3 class="text-lg font-bold text-gray-800 mb-2">
                {{ $t("message.titleForm") }}
              </h3>

              <div class="flex justify-between text-sm p-2 bg-gray-50 rounded-lg">
                <span class="text-gray-500">{{ $t("message.nameInputName") + ":" }}</span>
                <span class="font-medium text-gray-700">{{ values.name }}</span>
              </div>

              <div class="flex justify-between text-sm p-2 bg-gray-50 rounded-lg">
                <span class="text-gray-500">{{
                  $t("message.nameInputPhone") + ":"
                }}</span>
                <span class="font-medium text-gray-700">{{ values.phone }}</span>
              </div>

              <div class="flex justify-between text-sm p-2 bg-gray-50 rounded-lg">
                <span class="text-gray-500">{{
                  $t("message.nameInputTaxCode") + ":"
                }}</span>
                <span class="font-medium text-gray-700">{{ values.taxCode }}</span>
              </div>

              <div class="flex justify-between text-sm p-2 bg-gray-50 rounded-lg">
                <span class="text-gray-500">{{
                  $t("message.nameInputBudgetRelationshipCode") + ":"
                }}</span>
                <span class="font-medium text-gray-700">{{
                  values.budgetRelationshipCode
                }}</span>
              </div>

              <div class="flex justify-between text-sm p-2 bg-gray-50 rounded-lg">
                <span class="text-gray-500">{{
                  $t("message.nameInputCompanyName") + ":"
                }}</span>
                <span class="font-medium text-gray-700">{{ values.companyName }}</span>
              </div>

              <div class="flex justify-between text-sm p-2 bg-gray-50 rounded-lg">
                <span class="text-gray-500">{{
                  $t("message.nameInputCompanyAddress") + ":"
                }}</span>
                <span class="font-medium text-gray-700">{{ values.companyAddress }}</span>
              </div>
            </div>

            <!-- Tổng cộng -->
            <div
              class="flex justify-between text-xl font-bold text-gray-900 pt-4 border-t"
            >
              <span>{{ $t("message.totalPrice") + ":" }}</span>
              <span class="text-blue-600">{{ price.toLocaleString("vi-VN") }}đ</span>
            </div>
          </div>

          <!-- FOOTER -->
          <template #footer>
            <Button
              @click="OrderComplete()"
              rounded
              severity="info"
              :loading="loading"
              class="w-full"
              :label="$t('message.buttonOrderConfirmation')"
            />
          </template>
        </Dialog>

        <div v-for="product in carts" class="flex justify-content-between">
          <div class="flex justify-content-start">
            <img alt="food" :src="product.img" class="product-card p-2" />
            <div class="flex flex-column-reverse gap-3">
              <p>{{ (product.price * product.count).toLocaleString("vi-VN") }}đ</p>
              <p>
                {{ language === "vi" ? product.nameVi : product.nameEn }}
                {{ "x" + product.count }}
              </p>
            </div>
          </div>

          <div class="flex align-items-center gap-2">
            <Button
              rounded
              :disabled="checkDisable(product)"
              @click="removeCart(product)"
              icon="pi pi-minus m-2"
              severity="danger"
              variant="outlined"
            />

            <span class="m-2">{{ count(product) }}</span>

            <Button
              @click="addCart(product)"
              icon="pi pi-plus m-2"
              severity="primary"
              variant="outlined"
              :disabled="checkCount(product)"
              rounded
            />
            <Button
              @click="deleteCart(product)"
              icon="pi pi-trash m-2"
              severity="danger"
              variant="outlined"
              rounded
            />
          </div>
        </div>
      </template>
      <template #footer>
        <div v-if="!check" class="flex flex-column align-items-center">
          <span style="font-size: 2rem" class="pi pi-shopping-cart"></span>
          <p>{{ $t("message.titleCartEmpty") }}</p>
          <p>{{ $t("message.subTitleCartEmpty") }}</p>
        </div>
        <div class="flex justify-content-between">
          <p>{{ $t("message.totalPrice") + ":" }}</p>
          <p>{{ price.toLocaleString("vi-VN") }}đ</p>
        </div>
      </template>
    </Card>
  </div>

  <div v-if="check" class="flex justify-content-center">
    <Card class="m-2 w-7">
      <template #title>{{ $t("message.titleForm") }}</template>
      <template #content>
        <Form @submit="onSubmit">
          <div class="field mb-3">
            <label for="name">{{ $t("message.nameInputName") }}</label>
            <InputText
              id="name"
              :placeholder="$t('message.placeholderInputName')"
              class="w-full"
              v-model="name"
              v-bind="nameAttrs"
            />
            <span class="p-error"> {{ errors.name }}</span>
          </div>

          <div class="field mb-3">
            <label for="phone">{{ $t("message.nameInputPhone") }}</label>
            <InputText
              id="phone"
              class="w-full"
              v-model="phone"
              v-bind="phoneAttrs"
              :placeholder="$t('message.placeholderInputPhone')"
            />
            <span class="p-error"> {{ errors.phone }}</span>
          </div>

          <!-- Ghi chú -->
          <label>{{ $t("message.nameInputBill") }}</label>
          <div class="flex flex-wrap gap-4 p-2">
            <div class="flex items-center gap-2">
              <RadioButton value="noReceipt" v-model="getBill" />
              <label>{{ $t("message.nameRadioNoReceipt") }}</label>
            </div>
            <div class="flex items-center gap-2">
              <RadioButton value="getInvoice" v-model="getBill" />
              <label>{{ $t("message.nameRadioNoGetInvoice") }}</label>
            </div>
          </div>
          <div v-if="showBill()">
            <div class="field mb-3">
              <label for="taxCode">{{ $t("message.nameInputTaxCode") }}</label>
              <InputText
                id="taxCode"
                class="w-full"
                v-model="taxCode"
                v-bind="taxCodeAttrs"
                :placeholder="$t('message.placeholderTaxCode')"
              />
              <span class="p-error"> {{ errors.taxCode }}</span>
            </div>

            <div class="field mb-3">
              <label for="budgetRelationshipCode">{{
                $t("message.nameInputBudgetRelationshipCode")
              }}</label>
              <InputText
                id="budgetRelationshipCode"
                class="w-full"
                v-model="budgetRelationshipCode"
                v-bind="budgetRelationshipCodeAttrs"
                :placeholder="$t('message.placeholderBudgetRelationshipCode')"
              />
            </div>

            <div class="field mb-3">
              <label for="companyName">{{ $t("message.nameInputCompanyName") }}</label>
              <InputText
                id="companyName"
                class="w-full"
                v-model="companyName"
                v-bind="companyNameAttrs"
                :placeholder="$t('message.placeholderCompanyName')"
              />
            </div>

            <div class="field mb-3">
              <label for="companyAddress">{{
                $t("message.nameInputCompanyAddress")
              }}</label>
              <InputText
                id="companyAddress"
                class="w-full"
                v-model="companyAddress"
                v-bind="companyAddressAttrs"
                :placeholder="$t('message.placeholderCompanyAddress')"
              />
            </div>
          </div>
          <div class="flex justify-content-evenly align-center p-2">
            <Button
              @click="goToHome"
              class="col-5"
              label="Quay lại"
              severity="contrast"
              rounded
            />
            <Button
              rounded
              severity="info"
              class="col-5"
              label="Tiếp tục"
              type="submit"
            />
          </div>
        </Form>
      </template>
    </Card>
  </div>
</template>
<style>
.product-card {
  width: 100px;
  height: 100px;
  object-fit: cover;
  border-radius: 4px;
}
.p-error {
  color: red;
  font-size: 0.875rem;
}
</style>
