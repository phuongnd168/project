<template>
  <Toast />
  <ConfirmDialog></ConfirmDialog>

  <div class="card">
    <DataTable :value="products" tableStyle="min-width: 50rem">
      <template #header>
        <div class="flex justify-content-between">
          <label class="text-3xl font-bold text-blue-500">Danh sách sản phẩm </label>
          <Button
            label="Thêm"
            icon="pi pi-plus"
            severity="info"
            class="p-button-success p-button-sm"
            rounded
            @click="addProduct()"
          />
        </div>
      </template>

      <Column field="NameVi" header="Tên sản phẩm VN"></Column>
      <Column field="NameEn" header="Tên sản phẩm EN"></Column>
      <Column header="Ảnh">
        <template #body="slotProps">
          <img
            :src="`${slotProps.data.Img}`"
            :alt="slotProps.data.Img"
            class="w-24 rounded"
          />
        </template>
      </Column>
      <Column field="Price" header="Giá">
        <template #body="slotProps">
          {{ formatCurrency(slotProps.data.Price) }}
        </template></Column
      >
      <Column field="Quantity" header="Số lượng"></Column>
      <Column field="Categories" header="Danh mục">
        <template #body="{ data }">
          <MultiSelect
            v-model="data.CategoriesSelected"
            :options="categories"
            optionLabel="NameVi"
            filter
            placeholder="Select Categories"
            :maxSelectedLabels="2"
            class="w-100 truncate"
          />
        </template>
      </Column>
      <Column header="Thao tác">
        <template #body="{ data }">
          <div class="flex gap-2">
            <!-- Nút sửa -->
            <Button
              icon="pi pi-pencil"
              @click="updateProduct(data.Id)"
              class="p-button-sm p-button-rounded p-button-warn"
            />
            <!-- Nút xóa -->
            <Button
              icon="pi pi-trash"
              @click="confirmDelete(data.Id)"
              class="p-button-sm p-button-rounded p-button-danger"
            />
          </div>
        </template>
      </Column>
    </DataTable>
  </div>
</template>

<script setup>
import MultiSelect from "primevue/multiselect";
import { useRouter } from "vue-router";
import { ref, onMounted } from "vue";
import Column from "primevue/column";
import DataTable from "primevue/datatable";
import Button from "primevue/button";
import axios from "axios";
import ConfirmDialog from "primevue/confirmdialog";
import { useConfirm } from "primevue/useconfirm";
import { useToast } from "primevue/usetoast";
import Toast from "primevue/toast";
const confirm = useConfirm();
const toast = useToast();
const router = useRouter();

const products = ref([]);
const categories = ref([]);

onMounted(async () => {
  if (sessionStorage.getItem("addProduct")) {
    toast.add({
      severity: "success",
      summary: "Thành công",
      detail: "Thêm sản phẩm thành công",
      life: 3000,
    });
  }
  sessionStorage.removeItem("addProduct");
  try {
    const productsData = await axios.get(`http://localhost:5097/api/products`, {
      headers: {
        Accept: "application/json",
        "Content-Type": "application/json",
      },
    });

    if (productsData.data.StatusCode === 200) {
      const productsValue = await productsData.data;
      products.value = productsValue.Data;
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

    products.value.forEach((product) => {
      if (product.CategoryId && product.CategoryId.length > 0) {
        product.CategoriesSelected = categories.value.filter((c) =>
          product.CategoryId.some((pc) => pc === c.Id)
        );
      } else {
        product.CategoriesSelected = [];
      }
    });
  } catch (error) {
    console.error("Error fetching categories:", error);
    throw error;
  }
});
const formatCurrency = (value) => {
  return value.toLocaleString("vi-VN", {
    style: "currency",
    currency: "VND",
  });
};
const updateProduct = (id) => {
  router.push("/manager-products/update/" + id);
};
const confirmDelete = (id) => {
  confirm.require({
    message: "Bạn chắc chắn muốn xóa sản phẩm này?",
    header: "Xóa sản phẩm",
    icon: "pi pi-info-circle",
    rejectLabel: "Quay lại",
    rejectProps: {
      label: "Quay lại",
      severity: "secondary",
      outlined: true,
    },
    acceptProps: {
      label: "Xoá",
      severity: "danger",
    },
    accept: async () => {
      try {
        const result = await axios.delete(`http://localhost:5097/api/products/${id}`);
        if (result.data.StatusCode === 200) {
          products.value = products.value.filter((p) => p.Id !== id);
        }
      } catch (error) {
        console.error(error);
        alert("Xóa thất bại!");
      }
      toast.add({
        severity: "success",
        summary: "Thành công",
        detail: "Xóa sản phẩm thành công",
        life: 3000,
      });
    },
  });
};

const addProduct = () => {
  router.push("/manager-products/add");
};
</script>
