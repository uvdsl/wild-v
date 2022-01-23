<template>
  <div class="grid">
    <div class="col md:col-6 md:col-offset-3">
      <div class="p-inputgroup">
        <Button icon="pi pi-search" @click="fetchWorkflowFromURI" />
        <InputText
          placeholder="https://example.org/ex/workflow.ttl"
          v-model="workflow_baseIRI"
          @keyup.enter="fetchWorkflowFromURI"
        />
        <Button label="GET" @click="fetchWorkflowFromURI" />
      </div>
    </div>
  </div>

  <div class="grid">
    <div class="col">
      <div class="border sizing">
        <WorkflowVizzard
          :treeDataInput="treeData"
          :updateTreeData="updateTreeData"
          :key="treeData"
          @select="modifyData"
        />
      </div>
    </div>
    <div class="col">
      <Textarea v-model="ttl" @focusout="updateFromTTL" class="sizing" />
    </div>
  </div>

  <div>
    <SpeedDial
      :model="speedDialActions"
      type="circle"
      :radius="75"
      direction="left"
      showIcon="pi pi-ellipsis-h"
    />
  </div>

  <!-- :style="{ 
      position: 'absolute', 
      bottom: 0, 
      right: 0, 
      'margin': '10px',
      'margin-bottom': '13px'}" -->
  <Toast
    position="bottom-right"
    :breakpoints="{ '420px': { width: '100%', right: '0', left: '0' } }"
  />

  <ModificationDialog
    :key="showDialog"
    :showDialog="showDialog"
    :node="dataToModify"
    @returnValue="modified"
    @close="closeDialog"
  />
</template>

<script lang="ts">
import * as N3 from "n3";
import { RDF, WILD } from "@/lib/namespaces";
import { computed, defineComponent, ref, Ref, watch } from "vue";
import { useSolidSession } from "@/composables/useSolidSession";
import { toTTL } from "@/lib/n3Extensions";
import {
  deleteResource,
  getResource,
  parseToN3,
  putResource,
} from "@/lib/solidRequests";
import WorkflowVizzard from "@/components/workflowVizzard/WorkflowVizzard.vue";
import ModificationDialog from "@/components/workflowVizzard/ModificationDialog.vue";

import { useToast } from "primevue/usetoast";

export interface TreeData {
  label: string;
  type?: string;
  children: Array<TreeData>;
}

export default defineComponent({
  name: "Messenger",
  components: { WorkflowVizzard, ModificationDialog },
  setup() {
    const toast = useToast();
    const { authFetch } = useSolidSession();
    const speedDialActions = [
      {
        label: "Save",
        icon: "pi pi-save",
        command: () => {
          if (!isSaveable.value) {
            toast.add({
              severity: "error",
              summary: "Missing URI to save at!",
              detail: "Specify a HTTP-URI in the search bar.",
              life: 5000,
            });
            return;
          }
          putResource(workflow_baseIRI.value, ttl.value, authFetch.value)
            .then((resp) =>
              toast.add({
                severity: "success",
                summary: "Successful Save!",
                detail: "The workflow has been put at the URI.",
                life: 5000,
              })
            )
            .catch((err) =>
              toast.add({
                severity: "error",
                summary: "Error on save!",
                detail: err,
                life: 5000,
              })
            );
        },
      },
      {
        label: "Delete",
        icon: "pi pi-trash",
        command: () => {
          if (!isSaveable.value) {
            toast.add({
              severity: "error",
              summary: "Missing URI to delete!",
              detail: "Specify a HTTP-URI in the search bar.",
              life: 5000,
            });
            return;
          }
          deleteResource(workflow_baseIRI.value, authFetch.value)
            .then((resp) =>
              toast.add({
                severity: "warn",
                summary: "Successful Delete!",
                detail: "The resource has been deleted.",
                life: 5000,
              })
            )
            .catch((err) =>
              toast.add({
                severity: "error",
                summary: "Error on delete!",
                detail: err,
                life: 5000,
              })
            );
        },
      },
      // {
      //   label: "Link",
      //   icon: "pi pi-link",
      //   command: () => {
      //     toast.add({
      //       severity: "info",
      //       summary: "Not yet implemented!",
      //       detail: "Linking workflows is not supported yet.",
      //       life: 5000,
      //     });
      //   },
      // },
    ];

    const workflow_baseIRI = ref("");
    const isSaveable = computed(
      () =>
        workflow_baseIRI.value.startsWith("http://") ||
        workflow_baseIRI.value.startsWith("https://")
    );

    const _parseToN3 = (text: string, baseIRI: string) =>
      parseToN3(text, baseIRI).catch((err) => {
        toast.add({
          severity: "error",
          summary: "Error while parsing!",
          detail: err,
          life: 5000,
        });
        throw new Error(err);
      });

    const fetchWorkflowFromURI = async () => {
      if (
        !workflow_baseIRI.value.startsWith("http://") &&
        !workflow_baseIRI.value.startsWith("https://")
      ) {
        return;
      }
      const txt = await getResource(workflow_baseIRI.value, authFetch.value)
        .catch((err) => {
          toast.add({
            severity: "error",
            summary: "Error on fetch!",
            detail: err,
            life: 5000,
          });
          throw new Error(err);
        })
        .then((resp) => resp.text());

      const parsedN3 = await _parseToN3(txt, workflow_baseIRI.value);
      n3Store.value = parsedN3.store;
      n3Prefixes.value = parsedN3.prefixes;
    };

    const n3Store = ref(new N3.Store());
    const n3Prefixes: Ref<N3.Prefixes> = ref({});

    const updateFromTTL = async () => {
      const parsedN3 = await _parseToN3(ttl.value, workflow_baseIRI.value);
      // TODO isomorphism instead of relying on blank node naming...
      const dummyStore = new N3.Store();
      dummyStore.addQuads(parsedN3.store.getQuads(null, null, null, null));
      dummyStore.removeQuads(n3Store.value.getQuads(null, null, null, null));
      if (dummyStore.size === 0) {
        return;
      }
      n3Store.value = parsedN3.store;
      n3Prefixes.value = parsedN3.prefixes;
    };

    n3Prefixes.value["rdf"] = new N3.NamedNode(
      "http://www.w3.org/1999/02/22-rdf-syntax-ns#"
    );
    n3Prefixes.value["rdfs"] = new N3.NamedNode(
      "http://www.w3.org/2000/01/rdf-schema#"
    );
    n3Prefixes.value["xsd"] = new N3.NamedNode(
      "http://www.w3.org/2001/XMLSchema#"
    );
    n3Prefixes.value["wild"] = new N3.NamedNode("http://purl.org/wild/vocab#");
    n3Store.value.addQuad(
      new N3.NamedNode(workflow_baseIRI.value + "#model"),
      new N3.NamedNode(RDF("type")),
      new N3.NamedNode(WILD("WorkflowModel"))
    );
    n3Store.value.addQuad(
      new N3.NamedNode(workflow_baseIRI.value + "#model"),
      new N3.NamedNode(WILD("hasBehaviour")),
      new N3.NamedNode(workflow_baseIRI.value + "#root")
    );
    n3Store.value.addQuad(
      new N3.NamedNode(workflow_baseIRI.value + "#root"),
      new N3.NamedNode(RDF("type")),
      new N3.NamedNode(WILD("AtomicActivity"))
    );

    const visitChild = (
      child: N3.NamedNode | N3.Variable | N3.BlankNode,
      connectedQuads: N3.Store
    ) => {
      let quads = n3Store.value.getQuads(child, null, null, null);
      quads.forEach((quad: N3.Quad) => {
        if (
          !connectedQuads.has(quad) &&
          !(quad.object.termType === "Literal")
        ) {
          connectedQuads.addQuad(quad);
          visitChild(quad.object, connectedQuads);
        }
      });
    };

    const cleanN3 = () => {
      let connectedQuads = new N3.Store();
      let quads = n3Store.value.getQuads(
        null,
        WILD("hasBehaviour"),
        null,
        null
      );
      connectedQuads.addQuads(quads);
      quads.forEach((quad) => {
        visitChild(quad.subject, connectedQuads);
        if (quad.object.termType !== "Literal")
          visitChild(quad.object, connectedQuads);
      });
      n3Store.value.removeQuads(n3Store.value.getQuads(null, null, null, null));
      n3Store.value.addQuads(connectedQuads.getQuads(null, null, null, null));
    };

    const ttl = ref(
      toTTL(n3Store.value, n3Prefixes.value, workflow_baseIRI.value)
    );
    watch(n3Store, () => {
      ttl.value = toTTL(
        n3Store.value,
        n3Prefixes.value,
        workflow_baseIRI.value
      );
    }); //, { immediate: true });

    const translateToTreeData = () => {
      const workflow_node = n3Store.value.getSubjects(
        RDF("type"),
        WILD("WorkflowModel"),
        null
      )[0];
      const workflow = workflow_node.value.split("#");
      const root_node_cands = n3Store.value.getObjects(
        workflow_node,
        WILD("hasBehaviour"),
        null
      );
      // if (root_node_cands[0].termType !== "NamedNode") return
      const root_node = root_node_cands[0];
      return getNodeDataFromN3(root_node, n3Store.value);
    };
    const getNodeDataFromN3 = (
      node: N3.Quad_Object,
      n3Store: N3.Store
    ): TreeData => {
      // label
      const label =
        node.value.startsWith(workflow_baseIRI.value.split("#")[0]) &&
        workflow_baseIRI.value.split("#")[0] !== ""
          ? "#" + node.value.split("#")[1]
          : node.value;
      // type
      const type_node = n3Store.getObjects(node, RDF("type"), null);
      let type = undefined;
      if (type_node.length > 0) {
        //   type = type_node[0].value.startsWith(WILD(""))
        //     ? "wild:" + type_node[0].value.slice(27)
        type = type_node[0].value;
      }
      // children
      let children_nodes = n3Store.getObjects(
        node,
        WILD("hasChildActivity"),
        null
      );
      // get list of children
      const child_blank_node_array = n3Store.getObjects(
        node,
        WILD("hasChildActivities"),
        null
      );
      if (child_blank_node_array.length > 0) {
        let list_node = child_blank_node_array[0];
        let list_children = [];
        while (list_node.value !== RDF("nil")) {
          list_children.push(n3Store.getObjects(list_node, RDF("first"), null));
          list_node = n3Store.getObjects(list_node, RDF("rest"), null)[0];
        }
        children_nodes = children_nodes.concat(list_children.flat());
      }
      return {
        label: label,
        type: type,
        children: children_nodes.map((node) =>
          getNodeDataFromN3(node, n3Store)
        ),
      };
    };
    const treeData: Ref<TreeData> = ref(translateToTreeData());
    watch(n3Store, () => (treeData.value = translateToTreeData())); // , { immediate: true });

    //

    const showDialog = ref(false);
    const dataToModify = ref({});
    const updateTreeData = ref({});
    let oldLabel: string;

    const modifyData = (d: any /* lacking the correct d3 types here */) => {
      dataToModify.value = d;
      oldLabel = d.data.label;
      showDialog.value = true;
    };

    const modified = async (d: any) => {
      // closeDialog();
      // rename
      if (d.data.label !== oldLabel) {
        n3Store.value
          .getQuads(workflow_baseIRI.value + oldLabel, null, null, null)
          .forEach((quad) => {
            n3Store.value.addQuad(
              new N3.NamedNode(workflow_baseIRI.value + d.data.label),
              quad.predicate,
              quad.object,
              quad.graph
            );
            n3Store.value.removeQuad(quad);
          });
        n3Store.value
          .getQuads(null, null, workflow_baseIRI.value + oldLabel, null)
          .forEach((quad) => {
            n3Store.value.addQuad(
              quad.subject,
              quad.predicate,
              new N3.NamedNode(workflow_baseIRI.value + d.data.label),
              quad.graph
            );
            n3Store.value.removeQuad(quad);
          });
      }

      // type
      const old_type_query_result = n3Store.value.getQuads(
        workflow_baseIRI.value + d.data.label,
        RDF("type"),
        null,
        null
      );

      if (old_type_query_result.length === 0 && d.data.type !== undefined) {
        n3Store.value.addQuad(
          new N3.Quad(
            new N3.NamedNode(workflow_baseIRI.value + d.data.label),
            new N3.NamedNode(RDF("type")),
            new N3.NamedNode(d.data.type)
          )
        );
      }

      old_type_query_result.forEach((quad) => {
        if (
          quad.object.value.startsWith(WILD("")) &&
          d.data.type !== quad.object.value
        ) {
          n3Store.value.addQuad(
            new N3.Quad(
              new N3.NamedNode(workflow_baseIRI.value + d.data.label),
              old_type_query_result[0].predicate,
              d.data.type,
              old_type_query_result[0].graph
            )
          );
          n3Store.value.removeQuad(quad);
        }
      });

      // children
      const childQuads = new N3.Store();
      // get list of children
      const child_blank_node_array = n3Store.value.getQuads(
        new N3.NamedNode(workflow_baseIRI.value + oldLabel),
        WILD("hasChildActivities"),
        null,
        null
      );
      childQuads.addQuads(child_blank_node_array);

      if (child_blank_node_array.length > 0) {
        let list_node = child_blank_node_array[0].object;
        let list_children = [];
        while (list_node.value !== RDF("nil")) {
          const child_quad = n3Store.value.getQuads(
            list_node,
            RDF("first"),
            null,
            null
          );
          childQuads.addQuads(child_quad);
          list_children.push(child_quad[0].object);
          const list_node_quad = n3Store.value.getQuads(
            list_node,
            RDF("rest"),
            null,
            null
          );
          childQuads.addQuads(list_node_quad);
          list_node = list_node_quad[0].object;
        }
      }
      n3Store.value.removeQuads(childQuads.getQuads(null, null, null, null));

      let childText = `<${d.data.label}> <${WILD("hasChildActivities")}> (`;
      d.data.children.forEach((child: TreeData) => {
        if (child.type === undefined) {
          child.type = WILD("AtomicActivity");
          n3Store.value.addQuad(
            new N3.Quad(
              new N3.NamedNode(child.label),
              new N3.NamedNode(RDF("type")),
              new N3.NamedNode(WILD("AtomicActivity"))
            )
          );
        }
        childText += ` <${child.label}>`;
      });
      childText += " ) . ";
      if (d.data.children.length > 0) {
        await _parseToN3(childText, workflow_baseIRI.value).then((parsedN3) => {
          n3Store.value.addQuads(
            parsedN3.store.getQuads(null, null, null, null)
          );
        });
      }
      cleanN3();
      updateTreeData.value = d;
      ttl.value = toTTL(
        n3Store.value,
        n3Prefixes.value,
        workflow_baseIRI.value
      );
    };
    const closeDialog = () => {
      console.log("closed");
      showDialog.value = false;
    };

    return {
      speedDialActions,
      workflow_baseIRI,
      isSaveable,
      fetchWorkflowFromURI,
      ttl,
      updateFromTTL,
      treeData,
      modifyData,
      showDialog,
      dataToModify,
      updateTreeData,
      modified,
      closeDialog,
    };
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
.grid {
  margin: 5px;
}
// .p-inputgroup {
//   width: 100%;
// }
// TextArea {
// height: 100%;
// width: 100%;
// max-height: 100%;
// max-width: 100%;
//   margin-bottom: 10px;
// }
::v-deep() {
  .p-speeddial {
    bottom: 0;
    right: calc(50% - 2rem);
    padding-bottom: 15px;
    -webkit-tap-highlight-color: transparent;
  }
  .sizing {
    height: calc(100vh - 240px);
    width: 100%;
    max-height: calc(100vh - 240px);
    max-width: 100%;
  }
}
.border {
  border: 1px solid var(--surface-d);
  border-radius: 3px;
}
.border:hover {
  border: 1px solid var(--primary-color);
}
</style>
