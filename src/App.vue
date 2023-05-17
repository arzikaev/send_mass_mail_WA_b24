<template>
  <v-app>
    <v-app-bar app color="primary" dark>
      <div class="d-flex align-center"><h1>Отправка сообщений</h1></div>
    </v-app-bar>
    <v-main>
      <v-form>
        <v-container fluid>
          <v-row>
            <v-col>
              <v-select
                v-model="docTypeSelect"
                :items="docType"
                item-text="VALUE"
                item-value="ID"
                label="Тип компании"
                outlined
                multiple
              ></v-select>
            </v-col>
            <v-col>
              <v-text-field
                label="Название компании"
                v-model="title"
                outlined
              ></v-text-field>
            </v-col>
            <v-col>
              <v-autocomplete
                v-model="assignedSelect"
                :items="assignedList"
                item-text="VALUE"
                item-value="ID"
                label="Ответственный"
                outlined
              ></v-autocomplete>
            </v-col>
          </v-row>
          <v-btn
            color="success"
            class="mr-4"
            @click="getDeal"
            :disabled="loading"
          >
            Сформировать
          </v-btn>
          <v-btn
            color="orange"
            class="mr-4"
            @click="sendMail"
            :disabled="loading || companys.length <= 0"
          >
            Отправить сообщения
          </v-btn>
        </v-container>
      </v-form>
      <template v-if="loading === true">
        <v-progress-linear
          :active="loading"
          :indeterminate="loading"
          absolute
          top
          color="light-green darken-4"
          height="10"
        ></v-progress-linear>
      </template>
      <template v-if="companys.length > 0">
        <v-form>
          <v-container fluid>
            <v-row>
              <v-col cols="12" sm="6" md="4">
                <v-textarea
                  v-model="sendText"
                  label="Текст сообщения"
                  outlined
                ></v-textarea>
              </v-col>
            </v-row>
          </v-container>
        </v-form>
      </template>
      <v-card v-if="companys.length > 0">
        <v-data-table
          :headers="headers"
          :items="contacts"
          :items-per-page="5"
          class="elevation-1"
          no-data-text="Нет контактов"
          :footer-props="{
            'items-per-page-all-text': 'Все',
            'items-per-page-text': 'Контактов на стр.: ',
          }"
        >
          <template #[`item.actions`]="{ item }">
            <v-icon small @click="deleteItem(item)"> mdi-delete </v-icon>
          </template>
        </v-data-table>
      </v-card>
    </v-main>
  </v-app>
</template>

<script>
import axios from "axios";

export default {
  name: "App",
  data: () => ({
    loading: false,
    menu: false,
    assignedList: [],
    assignedSelect: "",
    dates: [],
    normalDate: [],
    title: "",
    docType: [
      { ID: "3", VALUE: "Дистрибьютор / Оптовик" },
      { ID: "8", VALUE: "Производитель" },
      { ID: "21", VALUE: "Торговые сети" },
      { ID: "20", VALUE: "КФХ" },
      { ID: "22", VALUE: "E-commece" },
      { ID: "23", VALUE: "Импортер" },
      { ID: "2", VALUE: "ОРЦ" },
      { ID: "18", VALUE: "Развитие торговли" },
      { ID: "1", VALUE: "Производители Новосибирска выгрузка FoodMarkets" },
      { ID: "UC_V7ZA27", VALUE: "HoReCa" },
      { ID: "UC_P1DKE8", VALUE: "Розничный магазин" },
      { ID: "UC_SF061B", VALUE: "Логистика" },
      { ID: "UC_GWB1XR", VALUE: "Юридические услуги" },
      { ID: "UC_NEHQ05", VALUE: "Поставщик услуг" },
      { ID: "UC_SX8XKP", VALUE: "Государственные органы" },
      { ID: "UC_FHM79R", VALUE: "Экспортер (импортер)" },
    ],
    docTypeSelect: "",
    parametrs: [],
    sendText: "",
    headers: [
      { text: "", value: "actions", sortable: false },
      { text: "ID", value: "id" },
      { text: "ФИО", value: "name" },
      { text: "Телефон", value: "phone" },
    ],
    companys: [],
    contacts: [],
    url: "",
  }),
  methods: {
    deleteItem(item) {
      //console.log({item})
      const deletIndex = this.contacts.indexOf(item);
      this.contacts.splice(deletIndex, 1);
    },
    async getDeal() {
      try {
        this.companys = [];
        this.loading = true;
        let i = 1;
        const params = {
          order: { DATE_CREATE: "ASC" },
          filter: {},
          select: ["ID"],
          start: 0,
        };
        if (this.title) params.filter["%TITLE"] = this.title;
        if (this.assignedSelect)
          params.filter.ASSIGNED_BY_ID = this.assignedSelect;
        if (this.docTypeSelect) params.filter.COMPANY_TYPE = this.docTypeSelect;
        const method = "crm.company.list";
        while (i > 0) {
          console.log({ params });
          const companysData = await axios.post(this.url + method, params);
          //console.log({dealsData})
          for (const company of companysData.data.result) {
            this.companys.push(company);
          }
          if (companysData.data.next) {
            params.start = companysData.data.next;
            //console.log(params.start)
          } else {
            i = 0;
          }
          //console.log(dealsData.data.next)
          await this.pause(200);
        }
        console.log(this.companys, "companys");
        const batchParamsGetContacts = {
          halt: 0,
          cmd: {},
        };
        const batchParamsGetContactToCompany = {
          halt: 0,
          cmd: {},
        };
        const contacts = [];
        const contactToCompany = [];
        const contactsIDs = [];
        for (const index in this.companys) {
          batchParamsGetContactToCompany.cmd[
            `contact_${index}`
          ] = `crm.company.contact.items.get?id=${this.companys[index].ID}`;
          if (index % 50 || index == this.companys.length - 1) {
            console.log({ batchParamsGetContactToCompany });
            const batchResult = await axios.post(
              this.url + "batch",
              batchParamsGetContactToCompany
            );
            console.log(batchResult.data.result.result, "batchRes");
            for (const elemBatchindex in batchResult.data.result.result) {
              for (const contact of batchResult.data.result.result[
                elemBatchindex
              ]) {
                contactToCompany.push(contact);
              }
              // console.log(batchResult.data.result.result[elemBatchindex])
            }
            batchParamsGetContactToCompany.cmd = {};
          }
          console.log({ contactToCompany });
        }

        for (const index in contactToCompany) {
          if (contactsIDs.indexOf(contactToCompany[index].CONTACT_ID) == -1) {
            contactsIDs.push(contactToCompany[index].CONTACT_ID);
            batchParamsGetContacts.cmd[
              `contact_${index}`
            ] = `crm.contact.get?id=${contactToCompany[index].CONTACT_ID}`;
            console.log()
            if (index % 50 || index == contactToCompany.length - 1) {
              console.log({ batchParamsGetContacts });
              const batchResult = await axios.post(
                this.url + "batch",
                batchParamsGetContacts
              );
              //console.log(batchResult.data.result.result, 'batchRes')
              for (const elemBatchindex in batchResult.data.result.result) {
                //console.log({elemBatchindex})
                if (
                  batchResult.data.result.result[elemBatchindex].HAS_PHONE ===
                  "Y"
                ) {
                  contacts.push({
                    id: batchResult.data.result.result[elemBatchindex].ID,
                    name: `${batchResult.data.result.result[elemBatchindex].LAST_NAME} ${batchResult.data.result.result[elemBatchindex].NAME} ${batchResult.data.result.result[elemBatchindex].SECOND_NAME}`,
                    phone:
                      batchResult.data.result.result[elemBatchindex]
                        .HAS_PHONE === "Y"
                        ? batchResult.data.result.result[elemBatchindex]
                            .PHONE[0].VALUE
                        : "",
                  });
                }
              }
              batchParamsGetContacts.cmd = {};
            }
          }
        }
        for (const contact of contacts) {
          if (this.contacts.map((e) => e.id).indexOf(contact.id) > -1) {
            //console.log(contact.id)
          } else {
            this.contacts.push(contact);
          }
        }
        this.loading = false;
      } catch (e) {
        this.loading = false;
        console.log(e);
      }
    },
    async sendMail() {
      try {
        this.loading = true;
        //console.log(this.sendText)
        //console.log(this.senderSelect)
        //console.log(this.deals)
        for (const contact of this.contacts) {
          const methodSend = "bizproc.workflow.start";
          const paramsSend = {
            TEMPLATE_ID: 399,
            DOCUMENT_ID: ["crm", "CCrmDocumentContact", contact.id],
            PARAMETERS: {
              text: this.sendText,
            },
          };
          const resultSend = await axios.post(
            this.url + methodSend,
            paramsSend
          );
          console.log({ resultSend });
          await this.pause(200);
        }
        console.log("SEND END");
        await window.location.reload();
      } catch (e) {
        console.log(e);
      }
    },
    async getFields() {
      try {
        console.log("getFields");
        const assignedData = await axios.post(this.url + "user.get", {
          FILTER: { ACTIVE: "Y" },
        });
        for (const assigned of assignedData.data.result) {
          this.assignedList.push({
            ID: assigned.ID,
            VALUE: `${assigned.LAST_NAME} ${assigned.NAME}`,
          });
        }

        const bizProcData = await axios.post(
          this.url + "bizproc.workflow.template.list ",
          {
            select: ["ID", "PARAMETERS"],
            filter: { ID: "399" },
          }
        );
        console.log(bizProcData.data.result, "bizProcData");
        //console.log(this.sender)
        this.sendText = bizProcData.data.result[0].PARAMETERS.text.Default;
      } catch (e) {
        console.log(e);
      }
    },
    pause(ms) {
      return new Promise((resolve) => {
        setTimeout(resolve, ms);
      });
    },
  },
  mounted() {
    this.getFields();
  },
};
</script>
