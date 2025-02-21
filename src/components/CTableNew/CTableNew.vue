<template>
  <div>
    <q-banner inline-actions class="q-mb-md text-black bg-listcolor">
      {{ tableCaption }}
      <template v-slot:action>
        <q-btn
          dense
          flat
          unelevated
          round
          color="primary"
          @click="expand = !expand" :icon="expand ? 'expand_less' : 'expand_more'"
        />
      </template>
    </q-banner>
    <form v-show="expand" @submit.prevent.stop="onSubmit" class="q-pb-md">
      <div v-if="selectedList.length > 0" class="row">
        <div class="list-group-item q-pa-md" 
          v-for="formElement in selectedList"
          :key="formElement.columnId"
          :class="formElement | classFormat">
          <div>
            <div 
              v-bind:class="{ 'required': !formElement.column.nullable}">
              {{formElement.column.caption}}:
            </div>

            <div class="row items-baseline content-center"
              style="border-bottom: 1px solid rgba(0,0,0,0.12)" 
              v-if="formElement.column.relationTableName">
              <div class="col-11">
                <span>{{ formElement.column.value | relationDataFormat(formElement.column) }}</span>
              </div>
              <div class="col-1">
                <q-btn round dense flat icon="zoom_in" 
                @click="openDialogClickAction(formElement.column)" />
              </div>
            </div>

            <q-input v-else-if="isStringType(formElement.column.dataType)"
              v-model="formElement.column.value"
              :placeholder="formElement.column.description"
              :type="formElement.isPwd ? 'password' : 'text'" >
              <template v-slot:append v-if="!formElement.isText" >
                <q-icon
                  :name="formElement.isPwd ? 'visibility_off' : 'visibility'"
                  class="cursor-pointer"
                  @click="formElement.isPwd = !formElement.isPwd"
                />
              </template>
            </q-input>

            <q-editor  v-else-if="isTextType(formElement.column.dataType)"
              v-model="formElement.column.value"
              :placeholder="formElement.column.description" >
            </q-editor>

            <q-input v-else-if="isDateTimeType(formElement.column.dataType)"
              v-model="formElement.column.value" >
              <template v-slot:prepend>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
                    <q-date v-model="formElement.column.value"
                    mask="YYYY-MM-DD HH:mm:ss"
                    @input="hideRefPopProxyAction('qDateProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>

              <template v-slot:append>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy ref="qTimeProxy" transition-show="scale" transition-hide="scale">
                    <q-time  v-model="formElement.column.value"
                    mask="YYYY-MM-DD HH:mm:ss"
                    format24h with-seconds
                    @input="hideRefPopProxyAction('qTimeProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

             <q-input v-else-if="isDateType(formElement.column.dataType)" 
              v-model="formElement.column.value">
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
                    <q-date  v-model="formElement.column.value"
                    mask="YYYY-MM-DD"
                    @input="hideRefPopProxyAction('qDateProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-input v-else-if="isTimeType(formElement.column.dataType)"
             v-model="formElement.column.value" >
              <template v-slot:append>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy ref="qTimeProxy" transition-show="scale" transition-hide="scale">
                    <q-time   v-model="formElement.column.value" 
                    mask="HH:mm:ss"
                    format24h with-seconds
                    @input="hideRefPopProxyAction('qTimeProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-toggle v-else-if="isBoolType(formElement.column.dataType)"
             v-model="formElement.column.value" >
            </q-toggle>

            <q-input 
              v-else-if="isNumberType(formElement.column.dataType)"
              v-model="formElement.column.value"
              :placeholder="formElement.column.description"
              type="number">
            </q-input>

            <CFile v-else-if="isAttachmentType(formElement.column.dataType)"
             v-model="formElement.column.value"
             @input="(data)=>{
              formElement.column.value = data.url;
             }"></CFile>

            <q-input v-else
              v-model="formElement.column.value"
              :placeholder="formElement.column.description"
              :type="formElement.isPwd ? 'password' : 'text'" >
              <template v-slot:append v-if="!formElement.isText" >
                <q-icon
                  :name="formElement.isPwd ? 'visibility_off' : 'visibility'"
                  class="cursor-pointer"
                  @click="formElement.isPwd = !formElement.isPwd"
                />
              </template>
            </q-input>
          </div>
        </div>
      </div>
      <div v-else>
        <div class="row items-baseline content-center justify-start q-pb-md"
            :key="item.name" v-for="item in insertColumns">
          <div class="col-3 text-right text-subtitle2 q-pr-md query-cond"
            v-bind:class="{ 'required': !item.nullable}">
          {{item.caption}}:</div>

          <div class="col-7">
            <div class="row items-baseline content-center"
              style="border-bottom: 1px solid rgba(0,0,0,0.12)" 
              v-if="item.relationTableName">
              <div class="col-11">
                <span>{{ item.value | relationDataFormat(item) }}</span>
              </div>
              <div class="col-1">
                <q-btn round dense flat icon="zoom_in" @click="openDialogClickAction(item)" />
              </div>
            </div>

            <div class="sql-box" v-else-if="isSqlType(item)">
              <CSqlEditor
                ref="sqleditor"
                :value="item.value"
                @changeTextarea="changeTextarea($event, item)">
              </CSqlEditor>
            </div>

            <q-input v-else-if="isDateTimeType(item.dataType)"
                v-model="item.value">
              <template v-slot:prepend>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
                    <q-date v-model="item.value"
                    mask="YYYY-MM-DD HH:mm:ss"
                    @input="hideRefPopProxyAction('qDateProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>

              <template v-slot:append>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy ref="qTimeProxy" transition-show="scale" transition-hide="scale">
                    <q-time v-model="item.value" mask="YYYY-MM-DD HH:mm:ss"
                    format24h with-seconds
                    @input="hideRefPopProxyAction('qTimeProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-input v-else-if="isDateType(item.dataType)"
                v-model="item.value">
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy ref="qDateProxy" transition-show="scale" transition-hide="scale">
                    <q-date v-model="item.value"
                    mask="YYYY-MM-DD"
                    @input="hideRefPopProxyAction('qDateProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-input v-else-if="isTimeType(item.dataType)"
                v-model="item.value">
              <template v-slot:append>
                <q-icon name="access_time" class="cursor-pointer">
                  <q-popup-proxy ref="qTimeProxy" transition-show="scale" transition-hide="scale">
                    <q-time v-model="item.value" mask="HH:mm:ss"
                    format24h with-seconds
                    @input="hideRefPopProxyAction('qTimeProxy')" />
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>

            <q-toggle v-else-if="isBoolType(item.dataType)"
              v-model="item.value">
            </q-toggle>

            <q-input
              v-else-if="isNumberType(item.dataType)"
              :placeholder="item.description"
              v-model="item.value"
              type="number" >
            </q-input>

            <CFile v-else-if="isAttachmentType(item.dataType)"
             v-model="item.value"
             @input="(data)=>{
              item.value = data.url;
             }"></CFile>

            <q-editor v-else-if="isTextType(item.dataType)"
              v-model="item.value"
              :placeholder="item.description" >
            </q-editor>

            <q-input
              v-else
              :placeholder="item.description"
              v-model="item.value"
              :type="item.isPwd ? 'password' : 'text'" >
              <template v-slot:append v-if="!item.isText" >
                <q-icon
                  :name="item.isPwd ? 'visibility_off' : 'visibility'"
                  class="cursor-pointer"
                  @click="item.isPwd = !item.isPwd"
                />
              </template>
            </q-input>
          </div>
        </div>
      </div>
    </form>
    <div :key="item.relationName" v-for="item in oneToOneMainToSubTables">
      <CTableNew
      :fkColumnName="item.fkColumnName"
      :ref="'rTableNewRef' + item.relationName"
      :dataSource="dataSource"
      :tableName="item.tableName" ></CTableNew>
    </div>

    <div :key="item.relationName" v-for="item in oneToManySubTables">
      <CTableList
      :fkColumnName="item.fkColumnName"
      :ref="'rTableListRef' + item.relationName"
      :dataSource="dataSource"
      :tableName="item.tableName"
       ></CTableList>
    </div>
  </div>
</template>

<style lang="stylus">
.required:before
  content: "* ";
  color: red;

.CodeMirror
  color: black;
  /* direction: ltr; */
  line-height: 22px;

.CodeMirror-hints
  z-index: 9999 !important;

.sql-box
  border: 1px solid #ddd;

</style>

<script>
import { tableService } from "../../service";
import { metadataTableService } from "../../service";
import { metadataRelationService } from "../../service";
import { extend } from 'quasar'
import CTableListReadDialog from '../../components/CTableListRead/CTableListReadDialog'
import { format } from 'sql-formatter';

export default {
  name: "CTableNew",
  props: {
    dataSource: {
      required: true
    },
    tableName: {
      required: true
    },
    fkColumnName: {
      required: false,
      type: String,
      default: "",
    }
  },

  data() {
    return {
      loading:true,
      expand: true,
      tableCaption: "",
      insertColumns: [],
      relationMap: {},
      oneToOneMainToSubTables: [],
      oneToManySubTables: [],
      selectedList: []
    };
  },

  created() {
    console.info("CTableNew created");
    this.init();
  },
  mounted: function() {
    console.info("CTableNew mounted");
  },
  activated: function() {
    console.info("CTableNew->activated");
  },
  deactivated: function() {
    console.info("CTableNew->deactivated");
  },
  updated: function() {
    console.info("CTableNew->updated");
  },
  destroyed: function() {
    console.info("CTableNew->destroyed");
  },
  filters: {
    relationDataFormat: function(value, item) {
       const dicFormat = function(v) {
          if (v) {
           if (item.relationDisplayColumns.length > 0) {
              let displayValues = [];
              item.relationDisplayColumns.forEach((t) => {
                  v[t.name] && displayValues.push(v[t.name]);
              });
              return displayValues.join("|");
           } else {
              return v[item.relationColumnName] || v;
           }
          }
          return null;
       }

       if (Array.isArray(value)) {
         let values = [];
         value.forEach((t) => {
            const subV = dicFormat(t);
            if (subV) {
              values.push(subV);
            }
         });

         return values.join(",");
       }

       if (value) {
         return dicFormat(value);
       }

       return null;
    },
    classFormat: function(formElement) {
      let value = "";
      if (formElement.width) {
        value = "col-" + formElement.width;
      }

      return value;
    },
  },
  computed: {},
  methods: {
    async init(tableName) {
      this.insertColumns = [];

      await this.loadMeta();
    },

    changeTextarea(val, item) {
      this.$set(item, "value", val);
    },

    isSqlType: function(item) {
      if (item.name.toUpperCase().indexOf("SQL") >= 0) {
        return true;
      } else {
        return false;
      }
    },

    isTextType: function(dataType) {
      if (dataType === "TEXT"
        || dataType === "LONGTEXT" ) {
        return true;
      } else {
        return false;
      }
    },

    isStringType: function(dataType) {
      if (dataType === "CHAR"
        || dataType === "VARCHAR" ) {
        return true;
      } else {
        return false;
      }
    },

    isNumberType: function(dataType) {
      if (dataType === "TINYINT"
        || dataType === "SMALLINT"
        || dataType === "MEDIUMINT"
        || dataType === "INT"
        || dataType === "BIGINT") {
        return true;
      } else {
        return false;
      }
    },

    isDateType: function(dataType) {
      if (dataType === "DATE") {
        return true;
      } else {
        return false;
      }
    },

    isTimeType: function(dataType) {
      if (dataType === "TIME") {
        return true;
      } else {
        return false;
      }
    },

    isDateTimeType: function(dataType) {
      if (dataType === "DATETIME") {
        return true;
      } else {
        return false;
      }
    },

    isBoolType: function(dataType) {
      if (dataType === "BOOL") {
        return true;
      } else {
        return false;
      }
    },

    isAttachmentType: function(dataType) {
      if (dataType === "ATTACHMENT") {
        return true;
      } else {
        return false;
      }
    },

    getData() {
      let data = {};
      for (let i = 0; i < this.insertColumns.length; i++) {
        const insertColumn = this.insertColumns[i];
        const columnName = insertColumn.name;

        const relation = this.relationMap[columnName];
        if (relation) {
          if (insertColumn.value) {
            if (insertColumn.multipleValue) {
              let valueArr = [];
              insertColumn.value.forEach((t) => {
                valueArr.push(t[insertColumn.relationColumnName]);
              });

              data[columnName] = valueArr.join(",");
            } else {
              data[columnName] = insertColumn.value[insertColumn.relationColumnName];
            }
          }

          data[relation.name] = insertColumn.value;
        } else {
          if (insertColumn.value != undefined
            && insertColumn.value != null
            && insertColumn.value.toString().trim() !== "") {
            data[columnName] = insertColumn.value;
          }
        }
      }

      this.oneToOneMainToSubTables.forEach((oneToOneMainToSubTable) => {
        const ref = this.$refs['rTableNewRef' + oneToOneMainToSubTable.relationName];

        const subData = ref[0].getData();
        data[oneToOneMainToSubTable.relationName] = subData;
      });

      this.oneToManySubTables.forEach((oneToManySubTable) => {
        const ref = this.$refs['rTableListRef' + oneToManySubTable.relationName];

        const subData = ref[0].getData();
        data[oneToManySubTable.relationName] = subData;
      });
      return data;
    },

    hideRefPopProxyAction(ref) {
      const proxys = this.$refs[ref];
      for (let i = 0; i < proxys.length; i++) {
        proxys[i].hide();
      }
    },

    openDialogClickAction(item) {
      this.$q.dialog({
        component: CTableListReadDialog,

        // optional if you want to have access to
        // Router, Vuex store, and so on, in your
        // custom component:
        parent: this, // becomes child of this Vue node
                      // ("this" points to your Vue component)
                      // (prop was called "root" in < 1.1.0 and
                      // still works, but recommending to switch
                      // to data: the more appropriate "parent" name)

        // props forwarded to component
        // (everything except "component" and "parent" props above):
        tableName: item.relationTableName,
        selectionProp: item.multipleValue ? 'multiple': 'single',
        data: item.value,
        dataSource: this.dataSource
      }).onOk((data) => {
        item.value = data;
      }).onCancel(() => {
        console.log('Cancel')
      }).onDismiss(() => {
        console.log('Called on OK or Cancel')
      });
    },

    async loadMeta() {
      const that = this;
      this.loading = true;
      try {
        const table = await metadataTableService.getByName(this.dataSource, this.tableName);
        this.tableCaption = table.caption;

        const tableRelations = await metadataRelationService.getByName(this.dataSource, this.tableName);

        /* 关联表元数据 */
        let relationMetadataMap = {};
        await Promise.all(tableRelations.map(async (tableRelation) => {
           if (tableRelation.relationType === "ManyToOne"
            || tableRelation.relationType === "OneToOneSubToMain") {
            const relationTable = await metadataTableService.getByName(this.dataSource, tableRelation.toTable.name);

            relationMetadataMap[tableRelation.toTable.name] = relationTable;
           }
        }));

        let relationMap = {};
        let oneToOneMainToSubTables = [];
        let oneToManySubTables = [];
        tableRelations.forEach((tableRelation) => {
          if (tableRelation.relationType === "OneToOneMainToSub") {
            oneToOneMainToSubTables.push({
              "relationName": tableRelation.name,
              "tableName": tableRelation.toTable.name,
              "fkColumnName": tableRelation.toColumn.name
            });
          } else if (tableRelation.relationType === "OneToMany") {
            oneToManySubTables.push({
              "relationName": tableRelation.name,
              "tableName": tableRelation.toTable.name,
              "fkColumnName": tableRelation.toColumn.name
            });
          } else if (tableRelation.relationType === "ManyToOne"
            || tableRelation.relationType === "OneToOneSubToMain") {
             const fromColumnName = tableRelation.fromColumn.name;
             relationMap[fromColumnName] = tableRelation;
           }
        });
        this.oneToOneMainToSubTables = oneToOneMainToSubTables;
        this.oneToManySubTables = oneToManySubTables;
        this.relationMap = relationMap;

        let insertColumns = [];
        for (let i = 0; i < table.columns.length; i++) {
          const column = table.columns[i];

          if (column.insertable) {
            const columnName = column.name;
            if (this.fkColumnName && columnName === this.fkColumnName) {
              continue;
            }

            if (column.defaultValue === "true") {
              column.value = true;
            } else if (column.defaultValue === "false") {
              column.value = false;
            } else {
              column.value = column.defaultValue;
            }

            if (column.dataType === 'PASSWORD') {
              column.isText = false;
              column.isPwd = true;
            } else {
              column.isText = true;
              column.isPwd = false;
            }

            if (column.dataType === 'TEXT'
              || column.dataType === 'LONGTEXT') { 
              if (!column.value) {
                column.value = "";
              }
            }

            const relation = this.relationMap[columnName];
            if (relation) {
              column.relationTableName = relation.toTable.name;
              column.relationColumnName = relation.toColumn.name;
              column.relationDisplayColumns= that.getDisplayableColumns(relationMetadataMap[column.relationTableName]);

              insertColumns.push(column);
            } else {
              insertColumns.push(column);
            }
          }
        }

        this.insertColumns = insertColumns;

        //formBuilder
        let query = {
          tableId:table.id
        };
        const formBuilders = await tableService.list(this.dataSource, "tableFormBuilder", 0, 999, null, query, null);
        let formBuilder = formBuilders.find(t => t.device === 'pc' 
          && t.type === 'create');
        let selectedList = [];
        if (formBuilder != null) {
          selectedList = JSON.parse(formBuilder.body);
          selectedList.forEach((formElement) => {
            formElement.column = table.columns.find(t => t.id === formElement.columnId);
          });
        }

        this.selectedList = selectedList;

        this.loading = false;
      } catch (error) {
        this.loading = false;
        console.error(error);
      }
    },

    getDisplayableColumns(tableMetadata) {
      const displayableColumns = tableMetadata.columns.filter(t => t.displayable === true);
      return displayableColumns;
    }
  }
};
</script>
