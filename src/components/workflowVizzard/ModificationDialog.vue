<template>
  <Dialog
    header="Header"
    v-model:visible="show"
    :modal="true"
    :draggable="false"
  >
    <template #header>
      <h3>Modify the Node!</h3>
    </template>
    <!-- Content -->
    <div class="grid">
      <b class="col-1">Label</b>
      <InputText class="col" v-model="node_modified.data.label" />
    </div>

    <div class="grid">
      <b class="col-1">Type</b>
      <span class="col flex flex-wrap justify-content-between">
      <div class="p-field-radiobutton col">
        <RadioButton
          id="Atomic"
          v-model="node_modified.data.type"
          :value="WILD('AtomicActivity col')"
        />
        <label for="Atomic"> Atomic</label>
      </div>
      <div class="p-field-radiobutton col">
        <RadioButton
          id="Interface"
          v-model="node_modified.data.type"
          :value="WILD('InterfaceActivity')"
        />
        <label for="Interface"> Interface</label>
      </div>
      <div class="p-field-radiobutton col">
        <RadioButton
          id="Conditional"
          v-model="node_modified.data.type"
          :value="WILD('ConditionalActivity')"
        />
        <label for="Conditional"> Conditional</label>
      </div>
      <div class="p-field-radiobutton col">
        <RadioButton
          id="Parallel"
          v-model="node_modified.data.type"
          :value="WILD('ParallelActivity')"
        />
        <label for="Parallel"> Parallel</label>
      </div>
      <div class="p-field-radiobutton col">
        <RadioButton
          id="Sequential"
          v-model="node_modified.data.type"
          :value="WILD('SequentialActivity')"
        />
        <label for="Sequential"> Sequential</label>
      </div>
      </span>
    </div>
    

    <div class="grid">
      <b class="col-1">Children</b>
      <div class="p-inputgroup col">
        <InputText
          placeholder="#childActivity"
          v-model="child_label_to_add"
          @keyup.enter="addChild"
        />
        <Button icon="pi pi-plus" @click="addChild" />
      </div>
    </div>

    <span
      class="p-col-offset-1 flex justify-content-between list-row"
      v-for="(child, index) in node_modified.data.children"
      :key="child.id"
    >
      <span> {{ child.label }} </span>
      <Button icon="pi pi-ban" @click="deleteChild(index)" />
    </span>

    <template #footer>
      <Button label="Save" icon="pi pi-check" iconPos="right" @click="submit" />
    </template>
  </Dialog>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, watch } from "vue";
import { WILD } from "@/lib/namespaces";
export default defineComponent({
  name: "ModificationDialog",
  props: ["showDialog", "node"],
  setup(props, { emit }) {
    const show = ref(props.showDialog);
    watch(show, () => {
      if (!show.value) emit("close", true);
    });

    const child_label_to_add = ref("");
    const node_modified = reactive(props.node);

    const addChild = () => {
      if (child_label_to_add.value == "") {
        return;
      }
      if (
        !child_label_to_add.value.startsWith("#") &&
        !child_label_to_add.value.startsWith("http://") &&
        !child_label_to_add.value.startsWith("https://")
      ) {
        child_label_to_add.value = "#" + child_label_to_add.value;
      }
      node_modified.data.children.push({
        label: child_label_to_add.value.trim(),
        children: [],
      });
      child_label_to_add.value = "";
    };

    const deleteChild = (index: Number) => {
      node_modified.data.children.splice(index, 1);
    };

    const submit = () => {
      addChild();
      console.log("### Dialog\t| Submits\n", node_modified);
      emit("returnValue", node_modified);
      show.value = false;
    };

    return {
      child_label_to_add,
      node_modified,
      show,
      addChild,
      deleteChild,
      submit,
      WILD
    };
  },
});
</script>

<style scoped>
.col {
  margin: 10px;
}
.col-1 {
  display: flex;
  align-items: center;
}
.list-row {
  padding-left: 15px;
  margin-right: 10px;
  margin-top: 5px;
  margin-bottom: 5px;
}
.p-field-radiobutton {
  white-space: nowrap;
}
</style>
