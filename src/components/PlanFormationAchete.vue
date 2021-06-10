<template>
  <div @mouseover="checkAllFormations">
    <h1> Liste des codes tarifaires </h1> <br/>
    <v-row class="bottom-space">
      <v-col
        cols="12"
        xs="12"
        lg="6"
        :key="index"
        v-for="(formation, key, index) in formations"
      >
      <v-card
          height="100%"
          :elevation="24"
          :outlined="false"
        >
        <v-card-subtitle>
          <div
            :key="key + index"
            v-for="(value, key, index) in $_.omit(formation, 'pk', 'Version', 'Description de fiche de formation', 'Duree', 'Code de fiche de formation')"
          >
            <span v-if="value!=='' && key=='Code tarifaire'"
              ><strong>{{ key }}:</strong> {{ value }}
            </span>

            <span v-if="value!=='' && key=='Libelle de fiche de formation'"
              ><strong>Description:</strong> {{ value }}
            </span>
          </div>
        </v-card-subtitle>
        <v-card-actions
            class="secondary"
            fixed
          >
            <v-btn
              color="success"
              tile
              :to="{ name: 'detailFormation', params: { formation: formation[currentNameColumnJoinWithSheetDetailsFormation]} }"
            >Consulter <v-icon>mdi-eye</v-icon>
            </v-btn>
          </v-card-actions>
      </v-card>
      </v-col>
    </v-row>
     <div>
        <v-btn
          color="black"
          text
          @click="showMailDialog=true"
        >
          Envoyer par mail
        </v-btn>
      </div>
      <v-dialog
      v-model="showMailDialog"
      max-width="600"
      persistent
    >
      <v-card>
        <v-card-title class="headline">Envoyer un mail</v-card-title>
        <v-card-text>
          <v-form ref="sendMail">
            <v-row
              dense
              row="12"
            >

              <v-col
                cols="12"
                ma-2
              >
                <v-combobox
                  v-model="chipsMail"
                  chips
                  counter
                  clearable
                  label="email"
                  multiple
                  solo
                >
                  <template v-slot:selection="{ attrs, item, select, selected }">
                    <v-chip
                      v-bind="attrs"
                      :input-value="selected"
                      close
                      @click="select"
                      @click:close="removeEmail(item)"
                    >
                      <strong>{{item}}</strong>&nbsp;
                    </v-chip>
                  </template>
                </v-combobox>
              </v-col>
              <v-col
                cols="12"
                ma-2
              >
                <v-text-field
                  label="Objet"
                  v-model="subject"
                  outlined
                  required
                >
                </v-text-field>
              </v-col>
              <v-col
                cols="12"
                ma-2
              >
                <v-textarea
                  label="Corps du message"
                  v-model="body"
                  required
                >
                </v-textarea>
              </v-col>
            </v-row>
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
            color="red darken-1"
            text
            @click="showMailDialog=false"
          >Annuler
          </v-btn>
          <v-btn
            v-if="!mailEmpty"
            color="green lighten-1 white--text"
            @click="sendByEmail"
          >Envoyer</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import PdfGenerator from '@/customClasses/pdfGenerator'

export default {
  data: () => ({
    showMailDialog: false,
    chipsMail: [],
    subject: '',
    body: '',
    selectedFormations: []
  }),

  computed: {
    ...mapGetters([
      'currentSheetId',
      'currentData',
      'isSignIn',
      'currentNameColumnJoinWithSheetDetailsFormation',
      'currentTemplate',
      'currentFormation',
      'currentGdocsTemplate',
      'currentMapping'
    ]),
    mailEmpty () {
      return this.$_.isEmpty(this.chipsMail)
    },
    formations () {
      let list = []

      if (Object.keys(this.currentData).length) {
        list = this.currentData['Fiches de formations']
      }
      return list
    }
  },
  methods: {
    removeEmail (item) {
      this.chipsMail.splice(this.chipsMail.indexOf(item), 1)
      this.chipsMail = [...this.chipsMail]
    },
    existValidMail () {
      let result = false
      this.chipsMail.forEach(mail => {
        // eslint-disable-next-line
        if (/^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/.test(mail)) {
          result = true
        }
      })
      return result
    },
    clearMail () {
      this.chipsMail = []
      this.body = ''
      this.subject = ''
      this.showMailDialog = false
    },
    sendByEmail () {
      if (this.selectedFormations.length > 0) {
        if (this.existValidMail()) {
          this.showMailDialog = false
          this.loading = true
          const promises = []
          this.selectedFormations.forEach(value => {
            const formation = this.currentFormation(value)
            this.chipsMail.forEach(mail => {
              // eslint-disable-next-line
              if (/^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/.test(mail)) {
                let objet = formation[this.currentMapping.labelFicheFormation.sheet][0][0][this.currentMapping.labelFicheFormation.column] + ' | ' + this.subject
                const pdfgen = new PdfGenerator(this.$gapi, this.currentGdocsTemplate, this.currentTemplate, formation, `${value}.pdf`, 'email', [{ adress: mail, message: this.body, objet: objet }])
                promises.push(pdfgen.generatePdf())
              }
            })
          })
          Promise.all(promises).then(
            () => {
              this.alert.message = 'Mails envoyés'
              this.alert.show = true
              this.alert.type = 'success'
              this.loading = false
              this.chipsMail = []
              this.body = ''
              this.subject = ''
              this.check = []
            }
          ).catch(err => {
            this.chipsMail = []
            this.body = ''
            this.subject = ''
            this.alert.message = err
            this.alert.show = true
            this.loading = false
            this.alert.type = 'error'
            this.check = []
          })
        } else {
          this.alert.message = 'Vous devez saisir une adresse mail valide'
          this.alert.show = true
          this.alert.type = 'warning'
        }
      } else {
        this.alert.message = 'Vous devez selectionner au moins une formation'
        this.alert.show = true
        this.alert.type = 'warning'
      }
    },
    checkAllFormations () {
      this.check = []
      this.formations.forEach((element, index) => {
        this.check.push(index)
        this.selectedFormations.push(element)
      })
    }
  }
}
</script>
