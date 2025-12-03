<template>
  <Toast />

  <Form @submit="onSubmit">
    <div class="p-6 max-w-2xl mx-auto bg-white shadow-lg rounded-xl">
      <h2 class="text-2xl font-bold text-yellow-600 mb-4">Sửa sản phẩm</h2>

      <div class="flex flex-col gap-4">
        <!-- Tên tiếng Việt -->
        <div>
          <label class="font-semibold">Tên sản phẩm (VN)</label>
          <InputText
            v-model="nameVi"
            v-bind="nameViAttrs"
            class="w-full mt-1"
            placeholder="Nhập tên sản phẩm tiếng Việt"
          />
          <span class="p-error"> {{ errors.nameVi }}</span>
        </div>

        <!-- Tên tiếng Anh -->
        <div>
          <label class="font-semibold">Tên sản phẩm (EN)</label>
          <InputText
            v-model="nameEn"
            v-bind="nameEnAttrs"
            class="w-full mt-1"
            placeholder="Nhập tên sản phẩm tiếng Anh"
          />
          <span class="p-error"> {{ errors.nameEn }}</span>
        </div>

        <!-- Giá -->
        <div>
          <label class="font-semibold">Giá</label>
          <InputNumber
            v-model="price"
            v-bind="priceAttrs"
            class="w-full mt-1"
            inputClass="w-full"
            mode="currency"
            currency="VND"
            locale="vi-VN"
            placeholder="Nhập giá sản phẩm"
          />
          <span class="p-error"> {{ errors.price }}</span>
        </div>

        <!-- Số lượng -->
        <div>
          <label class="font-semibold">Số lượng</label>
          <InputNumber
            placeholder="Nhập số lượng sản phẩm"
            v-model="quantity"
            v-bind="quantityAttrs"
            class="w-full mt-1"
            inputClass="w-full"
          />
          <span class="p-error"> {{ errors.quantity }}</span>
        </div>

        <!-- Danh mục -->
        <div>
          <label class="font-semibold">Danh mục</label>
          <div>
            <MultiSelect
              v-model="categoriesSelected"
              v-bind="categoriesSelectedAttrs"
              :options="categories"
              optionLabel="NameVi"
              optionValue="Id"
              filter
              placeholder="Chọn danh mục"
              :maxSelectedLabels="2"
              class="w-full truncate"
            />
          </div>
          <span class="p-error"> {{ errors.categoriesSelected }}</span>
        </div>

        <!-- Ảnh sản phẩm -->
        <div>
          <label for="img" class="font-semibold">Ảnh sản phẩm</label>
          <div>
            <input
              type="file"
              accept="image/*"
              @change="handleImageUpload"
              class="mt-1"
            />
          </div>

          <div v-if="previewImage" class="mt-3">
            <img :src="previewImage" class="w-40 rounded shadow" />
          </div>
          <span class="p-error">{{ errors.img }}</span>
        </div>

        <!-- Hành động -->
        <div class="flex justify-end gap-2 mt-4">
          <Button label="Quay lại" severity="secondary" outlined @click="preview()" />
          <Button type="submit" label="Sửa" severity="warn" :loading="loading" />
        </div>
      </div>
    </div>
  </Form>
</template>

<script setup>
import { useForm } from "vee-validate";
import * as yup from "yup";
import { Form } from "vee-validate";
import { reactive, ref } from "vue";
import InputText from "primevue/inputtext";
import InputNumber from "primevue/inputnumber";
import MultiSelect from "primevue/multiselect";
import Button from "primevue/button";
import axios from "axios";
import Toast from "primevue/toast";
import { useToast } from "primevue/usetoast";
import { useRouter, useRoute } from "vue-router";
import { onMounted } from "vue";
const check = ref(null);
const toast = useToast();
const CLOUD_NAME = "di0xgrb1e";
const UPLOAD_PRESET = "upload_preset";
const loading = ref(false);
const categories = ref([]);
const router = useRouter();
const route = useRoute();
const id = route.params.id;
const { defineField, errors, handleSubmit, setFieldValue } = useForm({
  validationSchema: yup.object({
    nameVi: yup.string().required(() => "Tên tiếng Việt không được để trống"),
    nameEn: yup.string().required(() => "Tên tiếng Anh không được để trống"),
    price: yup.string().required(() => "Giá không được để trống"),
    quantity: yup.string().required(() => "Số lượng không được để trống"),
    categoriesSelected: yup
      .array()
      .min(1, "Bạn phải chọn ít nhất 1 danh mục")
      .required("Bạn phải chọn ít nhất 1 danh mục"),
    img: yup
      .mixed()
      .nullable()
      .test("fileType", "Chỉ được phép upload ảnh", (value) => {
        if (!value) return true; // không chọn ảnh mới → OK
        if (typeof value === "string") return true; // ảnh cũ → OK
        return ["image/jpeg", "image/png", "image/gif"].includes(value.type);
      })
      .test("fileSize", "Ảnh phải nhỏ hơn 2MB", (value) => {
        if (!value) return true;
        if (typeof value === "string") return true;
        return value.size <= 2 * 1024 * 1024;
      }),
  }),
});
const [imgField, imgAttrs, imgMeta] = defineField("img");
const [nameVi, nameViAttrs] = defineField("nameVi");
const [nameEn, nameEnAttrs] = defineField("nameEn");
const [price, priceAttrs] = defineField("price");
const [quantity, quantityAttrs] = defineField("quantity");
const [categoriesSelected, categoriesSelectedAttrs] = defineField("categoriesSelected");
const previewImage = ref(null);

onMounted(async () => {
  const product = await axios.get(`http://localhost:5097/api/products/${id}`, {
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
    },
  });
  if (product.data.StatusCode === 200) {
    const { Data } = await product.data;
    if (Data) {
      check.value = true;
      nameVi.value = Data.NameVi;
      nameEn.value = Data.NameEn;
      price.value = Data.Price;
      quantity.value = Data.Quantity;
      categoriesSelected.value = Data.CategoryId;
      previewImage.value = Data.Img;
    }
  }

  const categoriesData = await axios.get(`http://localhost:5097/api/categories`, {
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
    },
  });
  if (categoriesData.data.StatusCode === 200) {
    const categoriesValue = await categoriesData.data;
    categories.value = categoriesValue.Data;
  }
});

const handleImageUpload = (e) => {
  const file = e.target.files[0] || null;

  setFieldValue("img", file);

  if (file) {
    check.value = false;
    previewImage.value = URL.createObjectURL(file);
  } else previewImage.value = null;
};

const onSubmit = () => {
  handleSubmit(
    async (values) => {
      loading.value = true;
      console.log(values);
      try {
        if (values.img?.name) {
          const formData = new FormData();
          formData.append("file", values.img);
          formData.append("upload_preset", UPLOAD_PRESET);
          const response = await fetch(
            `https://api.cloudinary.com/v1_1/${CLOUD_NAME}/image/upload`,
            {
              method: "POST",
              body: formData,
            }
          );
          const { secure_url } = await response.json();
          const result = await axios.put(`http://localhost:5097/api/products/${id}`, {
            headers: {
              Accept: "application/json",
              "Content-Type": "application/json",
            },

            nameVi: values.nameVi,
            nameEn: values.nameEn,
            price: values.price,
            quantity: values.quantity,
            listCategory: values.categoriesSelected,
            img: secure_url,
          });
          if (result.data.StatusCode === 200) {
            toast.add({
              severity: "success",
              summary: "Thành công",
              detail: "Sửa sản phẩm thành công",
              life: 3000,
            });
          }
        } else {
          const result = await axios.put(`http://localhost:5097/api/products/${id}`, {
            headers: {
              Accept: "application/json",
              "Content-Type": "application/json",
            },

            nameVi: values.nameVi,
            nameEn: values.nameEn,
            price: values.price,
            quantity: values.quantity,
            listCategory: values.categoriesSelected,
            img: previewImage.value,
          });
          if (result.data.StatusCode === 200) {
            toast.add({
              severity: "success",
              summary: "Thành công",
              detail: "Sửa sản phẩm thành công",
              life: 3000,
            });
          }
        }
      } catch (error) {
        console.error("Error fetching:", error);
        throw error;
      } finally {
        loading.value = false;
      }
    },
    (errs) => console.log("Errors:", errs)
  )();
};
const preview = () => {
  router.push("/manager-products");
};
</script>
