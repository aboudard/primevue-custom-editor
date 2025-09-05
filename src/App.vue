<template>
  <ThemeSwitcher />
  <div class="card">
    <DataTable
      v-model:editingRows="editingRows"
      :value="products"
      editMode="row"
      dataKey="id"
      @row-edit-save="onRowEditSave"
    >
      <Column field="code" header="Code" style="width: 20%">
        <template #editor="{ data, field }">
          <span>{{ data[field] }}</span>
        </template>
      </Column>
      <Column field="name" header="Name" style="width: 20%">
        <template #editor="{ data, field }">
          <InputText v-model="data[field]" fluid />
        </template>
      </Column>
      <Column field="inventoryStatus" header="Status" style="width: 20%">
        <template #editor="{ data, field }">
          <Select
            v-model="data[field]"
            :options="statuses"
            optionLabel="label"
            optionValue="value"
            placeholder="Select a Status"
            fluid
          >
            <template #option="slotProps">
              <Tag
                :value="slotProps.option.value"
                :severity="getStatusLabel(slotProps.option.value)"
              />
            </template>
          </Select>
        </template>
        <template #body="slotProps">
          <Tag
            :value="slotProps.data.inventoryStatus"
            :severity="getStatusLabel(slotProps.data.inventoryStatus)"
          />
        </template>
      </Column>
      <Column field="price" header="Price" style="width: 20%">
        <template #body="{ data, field }">
          {{ formatCurrency(data[field]) }}
        </template>
        <template #editor="{ data, field }">
          <InputNumber
            v-model="data[field]"
            mode="currency"
            currency="USD"
            locale="en-US"
            fluid
          />
        </template>
      </Column>
      <Column :rowEditor="true">
        <template #body="{ data, field, editorInitCallback }">
          <Button
            @click="onEditMode(editorInitCallback)"
            label="edit"
            size="small"
          />
          <Button
            @click="onDeleteMode(data, field)"
            label="delete"
            size="small"
          />
        </template>
        <template #editor="{ data, editorSaveCallback, editorCancelCallback }">
          <Button
            @click="onReadMode(editorSaveCallback)"
            :disabled="
              !data.name ||
              !data.inventoryStatus ||
              data.price === null ||
              data.price === ''
            "
            severity="secondary"
            label="save"
            size="small"
          />
          <Button
            @click="onCancelMode(editorCancelCallback, data)"
            severity="secondary"
            label="cancel"
            size="small"
          />
        </template>
      </Column>
    </DataTable>
    <div class="mt-3 flex justify-content-end">
      <Button
        @click="addNewRow"
        label="Add New Product"
        icon="pi pi-plus"
        class="mr-2"
      />
      <Button
        icon="pi pi-check"
        severity="success"
        :disabled="editingRows.length > 0"
        label="Validate"
      />
    </div>
  </div>
</template>

<script setup>
import { ProductService } from "@/service/ProductService";
import { onMounted, ref } from "vue";

const products = ref();
const editingRows = ref([]);
const newProductIds = ref(new Set()); // Track IDs of newly created products
const statuses = ref([
  { label: "In Stock", value: "INSTOCK" },
  { label: "Low Stock", value: "LOWSTOCK" },
  { label: "Out of Stock", value: "OUTOFSTOCK" },
]);

onMounted(() => {
  ProductService.getProductsMini().then((data) => (products.value = data));
});

const addNewRow = () => {
  const newId = Date.now().toString();
  const newProduct = {
    id: newId,
    code: `NEW${Date.now()}`,
    name: "",
    inventoryStatus: "",
    price: null,
  };

  // Track this as a new product
  newProductIds.value.add(newId);

  products.value.push(newProduct);

  // Add the new row to edit mode, keeping other rows in edit mode
  editingRows.value = [...editingRows.value, newProduct];
};

const onDeleteMode = (data, field) => {
  const index = products.value.findIndex((p) => p.id === data.id);
  if (index !== -1) {
    products.value.splice(index, 1);
    // Also remove from newProductIds if it was a new product
    newProductIds.value.delete(data.id);
  }
};

const onEditMode = (callback) => {
  callback();
};

const onCancelMode = (callback, rowData) => {
  // Only remove the row if it is a new product and is the one being canceled
  if (rowData && newProductIds.value.has(rowData.id)) {
    const index = products.value.findIndex((p) => p.id === rowData.id);
    if (index !== -1) {
      products.value.splice(index, 1);
    }
    newProductIds.value.delete(rowData.id);
  }
  callback();
};

const onReadMode = (callback) => {
  // If we successfully saved a new product, remove it from tracking
  const editingProduct = editingRows.value[0];
  if (editingProduct && newProductIds.value.has(editingProduct.id)) {
    newProductIds.value.delete(editingProduct.id);
  }
  callback();
};

const onRowEditSave = (event) => {
  let { newData, index } = event;
  products.value[index] = newData;
};
const getStatusLabel = (status) => {
  switch (status) {
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
const formatCurrency = (value) => {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(value);
};
</script>
