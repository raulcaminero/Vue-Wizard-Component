<template>
  <div class="wizard">
    <div class="navbar p-0">
      <div class="navbar-container">
        <ul class="nav nav-pills step-pills row mx-0 px-1">
          <wizard-step
            v-for="item in steps"
            :step-number="item.stepNumber"
            :number-desc="item.numberDesc"
            :step-desc="item.stepDesc"
            :active="item.active"
            :completed="item.completed"
            :key="item.stepNumber"
            :show-step="item.showStep"/>
        </ul>
      </div>
    </div>

    <div class="clearfix"></div>

    <section id="section-tabs" 
             class="w-100">
      <div class="tab-content card">
        <div class="tab-pane active">
          <div class="tab-pane-header">
            <div class="tag-badge ">
              <span class="dot text-center py-1"><i class="fas fa-user-circle orange-icon-define"></i></span>
              <span class="tag-badge-title py-1 pl-2">Definir PEI o PS</span>
            </div>
            <div class="label-current">
              <b>Paso {{ currentStep.numberDesc }}:</b>
              {{ currentStep.stepDesc }}
            </div>
          </div>
          <div class="tab-pane-body">
            <div id="errors-1" 
                 v-if="!isCurrentStepValid">
              <div v-for="message in messages"
                   :key="message.key"
                   class="alert alert-danger custom-red-alert"
                   role="alert"
                   :class="message.type">
                <a class="close" 
                   data-dismiss="alert">×</a>
                <i class="fa fa-times-circle" 
                   aria-hidden="true"></i>&nbsp;
                <span v-html="message.text"></span>
              </div>
            </div>

            <question-group
              v-for="step in steps"
              :key="step.stepNumber"
              :questions="step.questions"
              @onContinueClick="continueClicked"
              :step-type="step.stepType"
              :feature-icon="step.featureIcon"
              :confirmation-message="step.confirmationMessage"
              :confirmation-title="step.confirmationTitle"
              @statusChange="onStepStatusChange"
              :ref="'questionGroup_' + step.stepNumber"
              v-show="step.stepNumber == currentStepIndex"/>
          </div>
        </div>
      </div>
    </section>
    <div>
      <div>
        <div class="d-flex justify-content-end mr-5">
          <div class="row mt-3">
            <div class="col">
              <ActionButton variant="danger"
                            i="fa fa-times-circle"
                            @click="cancelClicked">
                Cancelar
              </ActionButton>
            </div>

            <div v-if="this.currentStepIndex > 1 && !IsShortForm"
                 class="col">
              <ActionButton variant="outline-success"
                            i="fa fa-arrow-circle-left"
                            @click="backClicked">
                Regresar
              </ActionButton>
            </div>

            <div class="col">
              <ActionButton variant="success"
                            i="fa fa-arrow-circle-right"
                            @click="btnContinueClick">
                Continuar
              </ActionButton>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import WizardStep from "./WizardStep";
import QuestionGroup from "./StepQuestionsGroup";
import router from "@/router/index";
import mixAlert from "@/mixin/alert.mixin";
import mixWait from "@/mixin/loading.mixin";
import Swal from "sweetalert2";
import EduEspYesNoNA from "@/utils/constants/EduEspYesNoNA";
import months from "@/utils/constants/months";
import asistantServiceOptions from "@/utils/constants/asistantServiceOptions";
import peiTypes from "@/utils/constants/pei/peiTypes";
import peiStatuses from "@/utils/constants/pei/peiStatus";
import peiTransitionAges from "@/utils/constants/pei/peiTransitionAges";
import moment from "moment";
import peiServices from "@/services/pei/general";
import ActionButton from "@/components/common/ActionButton";

const programTypes = Object.freeze({ PEI: 1, PEI_Unilateral: 2, PS: 3 });
const validationResultTypes = Object.freeze({ Good: 0, Error: 1, Warning: 2});
const messageTypes = Object.freeze({
  Warning: "alert-warning",
  Error: "alert-danger",
  Info: "alert-info",
  Success: "alert-success"
});
// const validationMessages = ["R03200050"];
const StudentStatuses = Object.freeze({
  PreRegistro: 125,
  Registro: 126,
  Evaluado: 127,
  Servido: 128,
  NoElegible: 129,
  Egreso: 131,
  Disponible: 200,
  Inactivo: 219,
  Elegible: 303
});

const PeiAdditionalQuestionType = Object.freeze({
  Evidencia_de_Participación_de_Alternatia_de_Ubicación: 1,
  Evidencia_de_Participación_Servicio_Transición_Post_Secundario: 2,
  Evidencia_de_que_la_Agencia_Proveyó_el_Servicio: 3,
  Fecha_de_Ubicación: 4,
  Asistente_de_Servicio: 5,
  Asistente_de_Servicio_Único: 6
});

export default {
  name: "PEIDefinitionWizard",
  mixins: [mixAlert, mixWait],
  props: {
    studentBirthDate: Date
  },
  components: {
    QuestionGroup,
    WizardStep,
    ActionButton
  },
  data() {
    //Dummy object con la ifo del estudiante
    let Student = {
      impedimento: "Otros Impedimentos de Salud", //proviene de la info del estudiante
      age: 7, //proviene de la info del estudiante
      matriculaActiva: false, //proviene de la info del estudiante
      NumberPeiPs: 0, // Numero de PEI o PS,
      HasPeioPSTothisAcademciYearSelected: false, // Tienee PEI o PS Firmado para el anos academico
      hasSignedPeiLastYear: false,
      ElegibilityInitialDate: new Date(2019, 10, 1),
      studentSie: 20951734
    };

    return {
      currentStepIndex: 1,
      messages: [
        {
          text: "Favor de validar y/o contestar las preguntas.",
          type: "alert-danger"
        }
      ],
      isCurrentStepValid: true,
      steps: [
        {
          stepNumber: 1,
          numberDesc: "1",
          stepDesc: "Año Academico",
          active: true,
          completed: false,
          isValid: true,
          showStep: true,
          questions: [
            {
              number: 1,
              displayNumber: "1",
              description: "Seleccione el año académico en el cual implementará el PEI o PS",
              answerControlType: 0,
              getOptions: this.getAcademicYears,
              onChange: this.onQuestion1ValueChange,
              isRequired: true,
              requireErrorCode: "3",
              requireErrorMessage: "",
              initialState: true
            },
            {
              number: 2,
              displayNumber: "2",
              description: "¿Este PEI o PS se considera para el año?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "2",
              requireErrorMessage: "",
              onChange: this.onQuestion2ValueChange,
              initialState: true
            },
            {
              number: 2.1,
              displayNumber: "2",
              subNumber: ".1",
              description: "¿Desea crear una copia de los datos ingresados para el PEI o PS?",
              answerControlType: 1,
              isVisible: false,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar.",
              initialState: false
            }
          ]
        },
        {
          stepNumber: 2,
          numberDesc: "P",
          stepDesc: "Preguntas Adicionales",
          showStep: false,
          validatePreviousYear: true,
          active: false,
          completed: false,
          isValid: true,
          questions: [
            {
              number: 1,
              displayNumber: "P1",
              description:
                "Según se seleccionó en el PEI o PS del año escolar " +
                (this.academicYear - 1) +
                " la alternativa de ubicación recomendada fue " +
                "[Aquí debe presentar la alternativa de ubicación seleccionada en el último PEI o PS en estatus firmado], " +
                " ¿se tiene evidencia de que el estudiante participó de esta alternativa de ubicación? ",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "190604",
              requireErrorMessage: "",
              onChange: this.onQuestion2Step1ValueChange,
              initialState: true
            },
            {
              number: 1.1,
              displayNumber: "P1",
              subNumber: ".1",
              description: "Seleccione la alternativa de ubicación en la que el estudiante participó",
              answerControlType: 0,
              getOptions: this.getAlternateLocation,
              isRequired: true,
              isVisible: false,
              requireErrorCode: "190605",
              requireErrorMessage: "",
              initialState: false
            },
            {
              number: 2,
              displayNumber: "P2",
              description:
                "<p>En la Parte V del PEI <b>" +
                this.academicYear +
                "</b> el estudiante tiene un programa de servicios con actividades dirigidas al proceso de transición post-Secundario</p>",
              answerControlType: 1,
              isVisible: false,
              onChange: this.onQuestion2Step2ValueChange,
              isRequired: true,
              requireErrorCode: "190606",
              requireErrorMessage: "",
              initialState: false
            },
            {
              number: 2.1,
              displayNumber: "P2",
              subNumber: ".1",
              description:
                "Favor de explicar las razones por la cual el estudiante no participó de las actividades dirigidas al proceso de transición post-Secundario",
              answerControlType: 3,
              isVisible: false,
              isRequired: true,
              requireErrorCode: "190607",
              requireErrorMessage: "",
              initialState: false,
              fields: [
                {
                  id: "comentario",
                  type: "textarea",
                  label: "<b>Comentario:</b>",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto"
                }
              ]
            },
            {
              number: 3,
              displayNumber: "P3",
              description:
                "En la Parte IX del PEI [Año Académico del último PEI o PS en estatus firmado] se estableció la necesidad de invitar a otras agencias para proveer algún servicio, ¿existe evidencia de que la agencia proveyó el servicio?",
              answerControlType: 1,
              isVisible: false,
              isRequired: true,
              requireErrorCode: "190608",
              requireErrorMessage: "",
              onChange: this.onQuestion3ValueChange,
              initialState: false
            },
            {
              number: 3.1,
              displayNumber: "P3",
              subNumber: ".1",
              description: "Especifique todas aquellas gestiones que se realizaron para que la agencia proveyera el servicio.",
              answerControlType: 3,
              isRequired: true,
              isVisible: false,
              requireErrorCode: "190609",
              requireErrorMessage: "",
              initialState: false
            },
            {
              number: 4,
              displayNumber: "P4",
              description: "¿El estudiante actualmente tiene el servicio de Asistente de Servicios?",
              answerControlType: 1,
              isRequired: true,
              isVisible: true,
              requireErrorCode: "190725",
              requireErrorMessage: "",
              onChange: this.onQuestion4ValueChange,
              initialState: true
            },
            {
              number: 4.1,
              displayNumber: "P4",
              subNumber: ".1",
              description: "¿Las tareas de este Asistente de Servicio están dirigidas a atender a un solo estudiante?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "190722",
              requireErrorMessage: "",
              onChange: this.onQuestion4AValueChange,
              isVisible: false,
              initialState: false
            },
            {
              number: 4.2,
              displayNumber: "P4",
              subNumber: ".2",
              description: "El servicio de Asistente es para: ",
              answerControlType: 0,
              getOptions: this.getAsistantServiceOptions,
              isRequired: true,
              requireErrorCode: "190724",
              requireErrorMessage: "",
              isVisible: false,
              initialState: false
            },
            {
              number: 4.3,
              displayNumber: "P4",
              subNumber: ".3",
              description: "Nombre del Asistente",
              answerControlType: 2,
              isRequired: true,
              requireErrorCode: "190723",
              requireErrorMessage: "",
              isVisible: false,
              initialState: false
            },
            {
              number: 5,
              displayNumber: "P5",
              description: "El estudiante está en estatus Elegible. Favor de seleccionar la fecha de ubicación inicial.",
              answerControlType: 4,
              isRequired: true,
              requireErrorMessage: "Favor de elegir fecha de ubicación inicial. ",
              pickerMinDate: this.pickerMinDate,
              isVisible: this.studentIsElegible,
              initialState: this.studentIsElegible
            },
            {
              number: 6,
              displayNumber: "P6",
              description:
                "Se identificó que en el PEI (anterior) el estudiante tení­a problemas de movilidad y acceso,\
                     el estudiante ¿Presentó alguna necesidad de acceso en la escuela?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion6ValueChange,
              isVisible: false,
              initialState: false,
              fields: [
                {
                  id: "comentario",
                  type: "textarea",
                  label: "<b>Comentario:</b>",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto",
                  showWithValue: false
                },
                {
                  id: "options",
                  label: "(Puede marcar más de una opción del siguiente catálogo):",
                  type: "multiple-selection-options",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  value: null,
                  options: [],
                  getOptions: this.getProblemasMovilidad,
                  selectedOptions: [],
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor seleccione una opción",
                  showWithValue: true
                }
              ]
            },
            {
              number: 6.1,
              displayNumber: "P6",
              subNumber: ".1",
              description: "¿Estas necesidades fueron expresadas en el PEI " + this.academicYear + "?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: false,
              initialState: false,
              onChange: this.onQuestion6AValueChange,
              notes: [
                {
                  message:
                    "Estas necesidades son expresadas en la Parte I(C)- Comentarios de salud, Parte III,(B) – Descripción de discapacidad, Parte VI(J) – Modificaciones al programa"
                }
              ],
              fields: [
                {
                  id: "comentario",
                  type: "textarea",
                  label: "Explique:",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto",
                  showWithValue: false
                }
              ]
            },
            {
              number: 6.2,
              displayNumber: "P6",
              subNumber: ".2",
              description: "¿Requirió un ajuste a la organización escolar para atender la necesidad de movilidad y/o acceso?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: false,
              initialState: false,
              onChange: this.onQuestion6BValueChange,
              fields: [
                {
                  id: "comentario",
                  type: "textarea",
                  label: "Explique:",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto",
                  showWithValue: false
                }
              ]
            },
            {
              number: 6.3,
              displayNumber: "P6",
              subNumber: ".3",
              description: "¿Se le proveyó el ajuste?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: false,
              initialState: false,
              onChange: this.onQuestion6CValueChange,
              fields: [
                {
                  id: "comentario",
                  type: "textarea",
                  label: "Explique:",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto",
                  showWithValue: false
                }
              ]
            },
            {
              number: 6.4,
              displayNumber: "P6",
              subNumber: ".4",
              description: "¿Qué tipo de ajuste se realizó?",
              answerControlType: 7,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Por favor seleccione una opción y/o introduzca texto",
              isVisible: false,
              initialState: false,
              fields: [
                {
                  id: "tipoajustes",
                  type: "multiple-selection-options",
                  label: "(selección múltiple)",
                  name: "tipo-ajustes",
                  class: "col-lg-12 col-sm-6",
                  value: null,
                  options: [],
                  getOptions: this.getTiposdeAjustesMovilidad,
                  selectedOptions: [],
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor seleccione una opción",
                  onChange: this.onQuestion6dot4FieldAjustesChange
                },
                {
                  id: "comentario",
                  type: "textarea",
                  label: "Explique:",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto",
                  visible: false
                }
              ]
            },
            {
              number: 6.5,
              displayNumber: "P6",
              subNumber: ".5",
              description: "¿El ajuste se proveyó en ambiente integrado y apropiado de acuerdo a sus necesidades?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Por favor seleccione una opción",
              isVisible: false,
              initialState: false,
              onChange: this.onQuestion6eValueChange
            },
            {
              number: 6.6,
              displayNumber: "P6",
              subNumber: ".6",
              description: "¿El ajuste fue realizado en un termino de 5 dí­as o menos a partir de la identificación de la necesidad?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: false,
              initialState: false,
              onChange: this.onQuestion6FValueChange,
              fields: [
                {
                  id: "comentario",
                  type: "textarea",
                  label: "Explique:",
                  name: "question-comment",
                  class: "col-lg-12 col-sm-6",
                  maxChars: 3000,
                  value: null,
                  required: true,
                  requireErrorCode: "",
                  requireErrorMessage: "Por favor introduzca texto",
                  showWithValue: false
                }
              ]
            }
          ]
        },
        {
          stepNumber: 3,
          numberDesc: "2",
          stepDesc: "Inicio",
          active: false,
          completed: false,
          isValid: true,
          showStep: true,
          questions: [
            {
              number: 7,
              displayNumber: "3",
              description:
                "¿<b>" +
               this.impedimentDescription +
                "</b>  es el impedimento por el cual el estudiante ha sido determinado elegible?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion7ValueChange,
              isVisible: true,
              initialState: true
            },
            {
              number: 8,
              displayNumber: "4",
              description: "¿El estudiante tiene la edad pertinente para cursar un grado pre-escolar?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion8ValueChange,
              isVisible: true,
              initialState: true
            },
            {
              number: 9,
              displayNumber: "5",
              description: "¿Se recomienda que el estudiante esté asignado a una escuela del DE para la implementación del PEI?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              warningMessage: "Warning test",
              onChange: this.onQuestion9ValueChange,
              isVisible: true,
              initialState: true
            },
            {
              number: 9.1,
              displayNumber: "5",
              subNumber: ".1",
              description: "¿Se recomienda que el estudiante esté ubicado en un Head Start, Hogar o Centro de Cuido?",
              answerControlType: 1,
              isVisible: false,
              initialState: false,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion9dot1ValueChange
            },
            {
              number: 9.2,
              displayNumber: "5",
              subNumber: ".2",
              description: "¿Se recomienda que el estudiante esté ubicado en una institución Educativa?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion9dot2ValueChange,
              isVisible: false,
              initialState: false
            },
            {
              number: 9.3,
              displayNumber: "5",
              subNumber: ".3",
              description: "Según sus contestaciones, ¿el estudiante estará ubicado en un colegio privado por decisión de sus padres (Unilateral)?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion9dot3ValueChange,
              isVisible: false,
              initialState: false
            },
            {
              number: 9.4,
              displayNumber: "5",
              subNumber: ".4",
              description: "¿Madre, Padre o Encargado rechaza que el DEPR realice una propuesta de ubicación pública?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion9dot4ValueChange,
              initialState: false,
              isVisible: false,
              fieldCaption: "Documento de Rechazo de Ubicación",
              fields: [
                {
                  id: "fecha",
                  type: "date",
                  label: "<b>Fecha:</b>",
                  name: "fecha",
                  class: "col-lg-3 col-sm-6",
                  value: null,
                  required: true,
                  requireErrorCode: "0737",
                  requireErrorMessage: "",
                  showWithValue: true,
                  maxDate: new Date()
                },
                {
                  id: "attachment",
                  attachmentId:null,
                  name: "attachment",
                  type: "file",
                  label: "<b>Formulario de Rechazo:</b>",
                  btnLabel: "Descargar",
                  class: "col-lg-6 col-sm-6",
                  value: null,
                  required: true,
                  requireErrorCode: "0738",
                  requireErrorMessage: "",
                  showWithValue: true
                }
              ]
            }
          ]
        },
        {
          stepNumber: 4,
          numberDesc: "3",
          stepDesc: "Detalle",
          active: false,
          completed: false,
          showStep: true,
          isValid: true,
          questions: [
            {
              number: 5.1,
              displayNumber: "6",
              description: "Favor de seleccionar la ubicación del estudiante recomendada para la implementación de este PEI o PS",
              answerControlType: 7,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "",
              isVisible: true,
              initialState: true,
              fields: [
                {
                  id: "region",
                  type: "dropdown",
                  label: "<b>Region:</b>",
                  name: "question-comment",
                  class: "col-lg-4 col-sm-6 mb",
                  maxChars: 100,
                  fieldkey: "Value",
                  fieldname: "Display",
                  onChange: this.onRegionChange,
                  options: [],
                  getOptions: this.getRegions,
                  value: null,
                  required: true,
                  requireErrorCode: "15",
                  requireErrorMessage: ""
                },
                {
                  id: "distrito",
                  type: "dropdown",
                  label: "<b>Distrito:</b>",
                  name: "question-comment",
                  class: "col-lg-4 col-sm-6 mb",
                  maxChars: 100,
                  fieldkey: "Value",
                  fieldname: "Display",
                  options: [],
                  onChange: this.onDistrictChange,
                  value: null,
                  required: true,
                  requireErrorCode: "17",
                  requireErrorMessage: ""
                },
                {
                  id: "escuela",
                  type: "dropdown",
                  label: "<b>Escuela:</b>",
                  name: "question-comment",
                  class: "col-lg-4 col-sm-6 mb",
                  maxChars: 100,
                  fieldkey: "Value",
                  fieldname: "Display",
                  options: [],
                  value: null,
                  required: true,
                  requireErrorCode: "19",
                  requireErrorMessage: ""
                }
              ]
            },
            {
              number: 6,
              displayNumber: "7",
              description: "Grado en el que recibirá acceso curricular ",
              answerControlType: 0,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar un grado.",
              getOptions: this.getGrados,
              onChange: this.onQuestion6Step4ValueChange,
              isVisible: true,
              initialState: true
            },
            {
              number: 7,
              displayNumber: "8",
              description: "¿El estudiante muestra conducta inapropiada?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 7.1,
              displayNumber: "8",
              description: "¿El estudiante tendrá servicios relacionados?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: this.isFromPeiDetail,
              initialState: false
            },
            {
              number: 8,
              displayNumber: "9",
              description: "¿El estudiante tendrá servicios de transición?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              onChange: this.onQuestion8Step4ValueChange,
              isVisible: true,
              initialState: true
            },
            {
              number: 8.1,
              displayNumber: "9",
              subNumber: ".1",
              description: "Seleccione el tipo de transición.",
              answerControlType: 0,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar el tipo de transición.",
              getOptions: this.getTransitions,
              isVisible: false,
              initialState: false
            },
            {
              number: 9,
              displayNumber: "10",
              description: "¿El estudiante tiene impedimento cognoscitivo significativo?",
              notes: [
                {
                  message:
                    "<b>SI</b> = En la evaluación psicométrica se evidencia que su funcionamiento cognitiva es significativamente bajo y/o muestra más de 6 años o más de rezago al compararse con estudiantes de su misma edad."
                },
                {
                  message:
                    "<b>No</b> = En la evaluación psicométrica se evidencia que su funcionamiento cognitiva es promedio o moderado al ser comparado con estudiantes de su misma edad."
                }
              ],
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 10,
              displayNumber: "11",
              description: "¿El estudiante necesitará de acomodos?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 11,
              displayNumber: "12",
              description: "Programa de Medición por el cual se evaluará el progreso académico y funcional del estudiante",
              notes: [
                {
                  message: "<b>Alterna</b> = Los estudiantes elegibles para medición alterna deben presentar impedimento cognoscitivo significativo."
                },
                {
                  message: "<b>N/A</b> = Estudiantes que reciben acceso al grado de K, 1ro, 2do, 9no, 10mo, 12mo y adultos."
                },
                {
                  message: "<b>Regular</b> = Todos los demás estudiantes de los grados 3, 4, 5, 6, 7, 8 y 11."
                }
              ],
              answerControlType: 0,
              getOptions: this.getMeasurePrograms,
              isRequired: true,
              onChange: this.onQuestion11Step4ValueChange,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar el programa de medición",
              isVisible: true,
              initialState: true
            },
            {
              number: 11.1,
              displayNumber: "12",
              subNumber: ".1",
              description: "¿El Programa de Medición será con Acomodo?",
              answerControlType: 1,
              isRequired: true,
              isVisible: false,
              initialState: false,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No."
            },
            {
              number: 12,
              displayNumber: "13",
              description: "¿Requiere el estudiante Servicios y Equipos de Asistencia Tecnológica?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 13,
              displayNumber: "14",
              description: "¿Recomienda servicios de año escolar extendido?",
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 14,
              displayNumber: "15",
              description: "¿El estudiante presenta un impedimento fí­sico que limita su movilidad y acceso?",
              notes: [
                {
                  message:
                    "<b>Sí</b> = Estudiante que presenta una limitación en su ambulación de forma temporal o permanente que requiere de alguna gestión para garantizar su acceso a la educación de forma equitativa."
                },
                {
                  message: "<b>No</b> = Estudiante que no presenta una necesidad de movilidad. "
                }
              ],
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 15,
              displayNumber: "16",
              description: "¿El estudiante requiere enseñanza en lenguaje de señas?",
              notes: [
                {
                  message:
                    "<b>Sí</b> = Estudiante que, por su impedimento auditivo, presenta una dificultad severa para comunicarse a través del lenguaje hablado por lo que requiere de aprendizaje en lenguaje de señas y/o adiestramiento en labio lectura."
                },
                {
                  message:
                    "<b>No</b> = Estudiante que no presenta problemas auditivos, estudiante que presenta problemas auditivos pero se comunica a través del lenguaje hablado."
                }
              ],
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            },
            {
              number: 16,
              displayNumber: "17",
              description: "¿El estudiante requiere enseñanza en Braille?",
              notes: [
                {
                  message:
                    "<b>Sí</b> = Estudiante cuenta con estos 3 criterios: <br />(1) tiene agudeza visual de 20/200 o menos en el mejor ojo con corrección y/o presenta una limitación en el campo visual, donde el ancho del diámetro esté opuesto a una distancia angular que no sea mayor de 20 grados, <br />(2) tiene una recomendación médica que estipula que tiene o tendrá un deterioro visual progresivo y <br />(3) muestra un funcionamiento cognitivo promedio o promedio bajo."
                },
                {
                  message:
                    "<b>No</b> = Estudiante que presenta una agudeza visual de 20/100 o mayor en el mejor ojo con corrección y/o presenta una limitación menor de 30 grados, su condición NO es progresiva y presenta impedimentos cognoscitivos moderados o significativos."
                }
              ],
              answerControlType: 1,
              isRequired: true,
              requireErrorCode: "",
              requireErrorMessage: "Favor de seleccionar Sí o No.",
              isVisible: true,
              initialState: true
            }
          ]
        },
        {
          stepNumber: 5,
          numberDesc: "4",
          stepDesc: "Creación",
          stepType: "message",
          featureIcon: "far fa-check-circle",
          confirmationTitle: "El PEI se comenzará a crear con la información detallada en las pantallas anteriores.",
          confirmationMessage: "¿Está seguro que desea continuar?",
          active: false,
          completed: false,
          showStep: true,
          isValid: true
        }
      ],
      recommendedLocation: "",
      studentSieId: 0,
      academicYear: 0,
      revisionType: "",
      programType: programTypes.PEI,
      peiDefinition: {
        academicYearId: 0
      },
      academicYears: null,
      validationMessages: [],
      Student: Student,
      impedimento: "Otros Impedimentos de Salud", //proviene de la info del estudiante
      studentAge: 7, //proviene de la info del estudiante
      matriculaActiva: false, //proviene de la info del estudiante
      signedPeiId: 0, // Numero de PEI o PS,
      hasPeioPSTothisAcademciYearSelected: false, // Tienee PEI o PS Firmado para el anos academico
      hasService: false, //
      RealAcademicYear: 0,
      PeiGeneralDB: {},
      PeiGeneral: {
        PeiId: "",
        StudentSieId: 0,
        PeiTypeCode: 0,
        ConfigAgeAmt: 0,
        ConfigAcademicYearId: 0,
        IsRevision: null,
        IsUnilateral: null,
        //propiedades paso adicional
        PeiAdditionalQuestions: [],
        //propiedades paso inicio
        ConfigImpedimentFlag: null,
        ConfigPrescholarFlag: null,
        ConfigAgeAmtFlag: null,
        ConfigPublicSchoolFlag: null,
        IsSpecialPrivateStudent: null,
        IsInstitutional: null,
        ConfigUnilateralFlag: null,
        ConfigLocationRejectFlag: null,
        LocationRejectDocument: null,
        //propidedades paso detalle
        ConfigLocationRegionId: null,
        ConfigLocationDistrictId: null,
        PeiLocationId: null,
        RegionId: null,
        DistrictId: null,
        ConfigSchoolId: null,
        GeneralStudentPublicSchoolFlag: null,
        ConfigGradeId: null,
        ConfigInapropriateConductFlag: null,
        ConfigTransitionServicesFlag: null,
        PeiTransitionCode: null,
        ConfigPlacementFlag: null,
        PeiOtherServMeasureCode: null,
        ConfigOtherServMeasurePlacementFlag: null,
        ConfigOtherServTechAssistEquipFlag: null,
        ConfigExtendedSchoolYearFlag: null,
        ConfigMobilityProblemFlag: null,
        ConfigSignsLanguageFlag: null,
        ConfigBrailleFlag: null,
        PeiUnilateralLocationId: null,
        ConfigUnilateralServiceEducationalFlag: null,
        ConfigUnilateralServiceRelatedFlag: null,
        ConfigImpedimentCognativeFlag: null,
        GeneralSchoolDistrictId: null,
        GeneralSchoolRegionId: null,
        GeneralSchoolId: null
      },
      PeiAdditionalQuestion: {
        PeiAdditionalQuestionId: null,
        PeiAdditionalQuestionAnswerId: null,
        AdditionalQuestionComment: null,
        AlternatePlacementId: null,
        AlternatePlacementDate: null,
        ServiceAssistantName: null,
        ServiceAssistantServiceId: null
      },
      SelectedPeiId: null,
      SelectedPeiYear: null,
      SelectedPeiYearId: null,
      SelectedPeiYearText: null,
      isFromTab1: false,
      IsShortForm: false,
      studentIsElegible: false,
      PickerMinDate: new Date(),
      impedimentDescription: ""
    };
  },
  computed: {
    currentStep() {
      return this.steps[this.currentStepIndex - 1];
    },
    nextStep() {
      return this.steps[this.currentStepIndex];
    },
    studentSei() {
      return this.$store.getters.SelectedStudent.studentId;
    },
    isFromPeiDetail() {
      return this.$route.params.peiId != undefined;
    }
  },
  async mounted() {
    this.ShowWait();
    await this.loadValidationMessages();
    await this.getStudentInfo(); 
    this.studentIsElegible = this.isElegible();
    this.pickerMinDate = this.formatDateForPicker(new Date(this.studentInfo.mipeStudent.actualEligibilityDate));
    this.impedimentDescription = this.studentInfo.mipeStudent.impedimentDescription;
    this.getQuestionElement(3,7).text =  "¿<b>" + this.impedimentDescription + "</b>  es el impedimento por el cual el estudiante ha sido determinado elegible?";
    
    if (this.isFromPeiDetail) {
      this.getStep(5).confirmationTitle = "El PEI o PS será modificado con la información detallada en las pantallas anteriores.";

      this.SelectedPeiId = this.$route.params.peiId;
      await this.getPeiYear();
      await this.getPei();
      await this.toPeiDetail();

      if (this.PeiGeneralDB.unilateralFlag == "0") {
        this.isFromTab1 = true;
        await this.mapPeiAnswersToQuestions();
      }else{
        await this.mapPSAnswersToQuestions();
      }
    }
    this.HideWait();
  },

  methods: {
    async getStudentInfo() {
      let self = this;
      await self.$ApiGet("/api/student/getstudentdetails?studentSie=" + this.$store.state.selectedStudent.studentId, null, true)
        .then(res => {
          this.studentInfo = res.data;
        })
        .catch(err => {
          this.ShowToast("Error", err.message, "error");
        })

    },
    async getPei() {
      let self = this;
      await self.$ApiGet("/api/pei/getpei?peiId=" + this.SelectedPeiId, null, true)
        .then(res => {
          self.PeiGeneralDB = res.data;
          if (self.PeiGeneralDB.generalClosedDate) {
            self.PeiGeneralDB.generalClosedDate = new Date(self.model.generalClosedDate);
          }
          if (self.PeiGeneralDB.unilateralFlag == "1" && self.PeiGeneralDB.peiStatusCode == peiStatuses.Borrador) {
              self.IsShortForm = true;
          }
        })
        .catch(err => {
          self.response = err;
        });
    },
    async onRegionChange(item) {
      if (item && item.Value) {
        let academicyear = !this.isFromPeiDetail
          ? this.getQuestionElement(1, 1).selectedAnswer
          : this.SelectedPeiYear
          ? this.SelectedPeiYear
          : this.GetCurrentYear();
        let questionElement = this.getQuestionElement(4, 5.1);
        let distritoField = questionElement.getField("distrito");
        let escuelaField = questionElement.getField("escuela");
        distritoField.value = null;
        escuelaField.value = null;
        distritoField.options = await this.getDistricts(academicyear, item.Value);
      }
    },
    async onDistrictChange(item) {
      if (item && item.Value) {
        let questionElement = this.getQuestionElement(4, 5.1);
        let escuelaField = questionElement.getField("escuela");

        let isPublic = this.getQuestionElement(3, 9).selectedAnswer;
        let isSpecialSchool = this.getQuestionElement(3, 9.1).selectedAnswer;
        let isInstitution = this.getQuestionElement(3, 9.2).selectedAnswer;
        let selectedAnswerYearId = this.getQuestionElement(1, 1).selectedAnswerId;    
        let isUnilateral = this.$store.state.pei.configUnilateralFlag;

        escuelaField.value = null;
        escuelaField.options = await this.getSchools(selectedAnswerYearId, item.Value, isUnilateral, isPublic, isSpecialSchool, isInstitution);
      }
    },
    loadValidationMessages() {
      const messageCodes = [
        "2",
        "3",
        "4",
        "R03200049",
        "R03200050",
        "190604",
        "190605",
        "190608",
        "190609",
        "190725",
        "190722",
        "190724",
        "190723",
        "22061",
        "190902",
        "190606",
        "190607",
        "15",
        "17",
        "19",
        "0737",
        "0738"
      ];
      let promise = this.$ApiGet("/api/validationsmessages?messageCodes=" + messageCodes.join("&messageCodes="));
      promise.then(result => {
        this.validationMessages = result.data;

        //set messages for all question
        for (let stepIdx = 0; stepIdx < this.steps.length; stepIdx++) {
          const step = this.steps[stepIdx];
          if (step.questions != undefined) {
            step.questions.forEach(question => {
              if (question.requireErrorCode != "" && question.requireErrorCode != null && question.requireErrorCode != undefined) {
                question.requireErrorMessage = this.getValidationMessage(question.requireErrorCode);
              }
              if (question.fields != null && question.fields.length > 0) {
                question.fields.forEach(item => {
                  let questionElement = this.getQuestionElement(step.stepNumber, question.number);
                  let field = questionElement.getField(item.id);
                  if (field.requireErrorCode != "" && field.requireErrorCode != null && field.requireErrorCode != undefined) {
                    field.requireErrorMessage = this.getValidationMessage(field.requireErrorCode);
                  }
                });
              }
            });
          }
        }
      });
    },
    getValidationMessage(code) {
      let result = this.validationMessages.find(item => item.id === code);
      return result.name;
    },
    showMessage(messageText, type) {
      this.messages.push({
        text: messageText,
        type: type
      });
    },
    onStepStatusChange(stepElement) {
      this.isCurrentStepValid = stepElement.isValid;
    },
    async getAlternateLocation() {
      const result = await this.getAlternateLocations();

      let alternate = [];
      result.forEach(element => {
        alternate.push({
          Value: element.codeNo,
          Display: element.spashortDescription,
          isSelected: false
        });
      });

      return alternate;
    },
    getAcademicYears() {
      let promise = new Promise(async resolve => {
        let res = await this.$ApiGet("/api/academicyears/peiquestions");
        let academicYears = [];
        res.data.forEach(element => {
          academicYears.push({
            Id: element.id,
            Value: element.year,
            Display: element.description,
            isSelected: false
          });
        });
        resolve(academicYears);
      });
      return promise;
    },
    getAsistantServiceOptions() {
      return this.resolveOptions(asistantServiceOptions);
    },
    getProblemasMovilidad() {
      let options = [
        {
          option: "Necesidad de tener acceso a la escuela (entradas a la escuela)"
        },
        {
          option: "Necesidad de tener acceso al edificio donde recibió clases"
        },
        {
          option: "Necesidad de tener acceso al (los) salón(es) de clases donde fue asignado, salón de educación especial o área de terapias"
        },
        {
          option: "Necesidad de tener acceso a áreas comunes (comedor, biblioteca, baños, laboratorios o el patio)"
        },
        {
          option: "Necesidad de hacer ajustes dentro del salón de clases para facilitar su movilidad."
        }
      ];
      return this.resolveOptions(options);
    },
    getTiposdeAjustesMovilidad() {
      let options = [
        { option: "El grupo se movió a la planta baja " },
        { option: "Se realizó un cambio de salón" },
        {
          option: "Se ofrecieron acomodos razonables en el área de ambiente y lugar"
        },
        { option: "Se cambió de horario del Programa del Maestro" },
        {
          option: "Se solicitó recursos adicionales (Ej. Asistente de servicios para movilidad, Maestro para orientación y movilidad)"
        },
        {
          option: "Se hizo una reducción de la matrí­cula en el grupo"
        },
        { option: "Se requirió trasladar estudiante a otra escuela." },
        {
          option: "Se requirió ubicar a un estudiante en el hogar u hospitales."
        },
        { option: "Otros." }
      ];
      return this.resolveOptions(options);
    },
    resolveOptions(options) {
      let promise = new Promise(async resolve => {
        let optionsResponse = [];
        options.forEach(element => {
          optionsResponse.push({
            Value: element.value ? element.value : element.option,
            Display: element.option,
            isSelected: false
          });
        });
        resolve(optionsResponse);
      });
      return promise;
    },
    async cancelClicked() {
      let message = "¿Está seguro que desea cancelar la creación del PEI o PS?";
      if(this.isFromTab1){
          message = "¿Está seguro que desea cancelar la actualización del PEI o PS?";
      }
      const answer = await mixAlert.methods.ShowCancelModal("Atención", message, "Sí", "No");
      if (answer) {
        if(this.isFromTab1)
          router.push({ path: "/pei/detail" });
        else
          router.push({ path: "/pei/history" });
      }
    },
    async btnContinueClick() {
      let currentQuestionGroup = this.$refs["questionGroup_" + this.currentStepIndex][0];
      if (currentQuestionGroup.validate()) {
        await currentQuestionGroup.continue();
      }
    },
    async continueClicked(questionGroup) {
      if (questionGroup.isValid) {
        let question2dot1 = this.getQuestionElement(1, 2.1);
        let nextStepIndex = 0;

        // si es en el paso de inicio y tiene seleccionado hacer copia de PEI
        if (this.currentStepIndex == 1) {
          if (question2dot1.selectedAnswer) {
            this.toWizardLastStep();
          }
        }
          
          //si hacemos click en el paso de creación
          if(this.currentStepIndex == this.steps.length){ 

            //se crea copia de pei
            if(question2dot1.selectedAnswer){  
              if(this.PeiGeneral.ConfigUnilateralFlag){
               await this.createPSCopy();
               return;
              }            
              await this.createPeiCopy();
              return;
          }

          this.mapQuestionsAnswers();

          // si tenemos peiId desde el router es porque venimos de la pantalla detalle PEI y vamos a actualizar
          // en caso contrario es un Pei nuevo
          let peiId = this.$route.params.peiId;
          if (peiId) {
            this.PeiGeneral.PeiId = peiId;
            await this.updatePei();
          } else {
            await this.createPei();
          }
        } else if (this.currentStepIndex < this.steps.length && this.currentStep.isValid) {
          if (this.nextStep.validatePreviousYear) {
            this.nextStep.showStep = await this.hasSignedPei(this.studentSei, this.RealAcademicYear - 1);
          }

          let step4 = this.getStep(4);

          let question5dot4Element = this.getQuestionElement(3, 9.4);

          if (this.currentStep.stepNumber == 3 && question5dot4Element.selectedAnswer && question5dot4Element.isVisible) {
            nextStepIndex = this.steps.length;
            step4.showStep = false;
          } else if (this.currentStepIndex < this.steps.length && this.currentStep.isValid) {
            let peiId = this.studentInfo.mipeStudent.peiId;
            let result = await this.getRecommendedLocation(peiId);
            this.recommendedLocation = result[0] && result[0].description;

            if (this.nextStep.showStep || this.nextStep.stepNumber == 4) {
              if (this.nextStep.stepNumber == 2 && !this.getQuestionElement(2, 2).isVisible && !this.isFromPeiDetail) {
                await this.hasAgencyLastYearSignedPei();
              }

              let question1Step2Element = this.getQuestionElement(2, 1);
              question1Step2Element.text =
                "<p>Según se seleccionó en el PEI o PS del año escolar <b>" +
                (this.RealAcademicYear - 2) +
                "-" +
                (this.RealAcademicYear - 1) +
                "</b> la alternativa de ubicación recomendada fue <b>" +
                this.recommendedLocation +
                "</b>, " +
                " ¿se tiene evidencia de que el estudiante participó de esta alternativa de ubicación? </p>";

              nextStepIndex = this.currentStepIndex + 1;

              step4.showStep = true;
            } else {
              //Jump one step
              nextStepIndex = this.currentStepIndex + 2;
              step4.showStep = true;
            }
          }
          this.setCurrentStep(nextStepIndex);
        }
      }
    },
    setCurrentStep(stepIndex) {
      this.currentStep.active = false;
      this.currentStep.completed = true;
      this.currentStepIndex = stepIndex;
      this.currentStep.active = true;
    },
    backClicked() {
      let question2dot1 = this.getQuestionElement(1, 2.1);

      // si es en el paso de inicio y tiene seleccionado hacer copia de PEI
      if (this.currentStepIndex == 5) {
        if (question2dot1.selectedAnswer) {
          this.toWizardFirstStep();
        }
      }

      let question5dot4Element = this.getQuestionElement(3, 9.4);
      let step4 = this.getStep(4);

      if (this.currentStep.stepNumber == this.steps.length && question5dot4Element.selectedAnswer) {
        this.currentStep.active = false;
        this.currentStep.completed = false;
        this.currentStepIndex = this.steps.length - 2;
        this.currentStep.active = true;
        step4.showStep = false;
      } else {
        if (this.steps[this.currentStepIndex - 2].showStep) {
          this.currentStep.active = false;
          this.currentStep.completed = false;
          this.currentStepIndex--;
          this.currentStep.active = true;
        } else {
          this.currentStep.completed = false;
          this.currentStep.active = false;
          this.currentStepIndex = this.currentStepIndex - 2;
          this.currentStep.active = true;
        }
        step4.showStep = true;
      }
    },
    async getPeiType(studentSei, academicYearId) {
      return await new Promise(resolve => {
        this.$ApiGet("/api/pei/getpeitype?studentSieId=" + studentSei + "&academicYearId=" + academicYearId)
          .then(res => {
            if (res && res.data != null) {
              if (res.data == 4) {
                resolve("El primero");
              } else {
                resolve("Revisión");
              }
            }
          })
          .catch(err => {
            this.response = err;
          });
      });
    },
    async getPeiYear() {
      return await this.$ApiGet("/api/pei/getpeiYear?peiId=" + this.SelectedPeiId)
        .then(res => {
          this.SelectedPeiYear = res.data.year;
          this.SelectedPeiYearId = res.data.id;
          this.SelectedPeiYearText = res.data.description;
        })
        .catch(err => {
          this.response = err;
        });
    },
    async hasSignedPei(studentSei, year) {
      return await new Promise(resolve => {
        this.$ApiGet("/api/pei/hassignedpei?studentSieId=" + studentSei + "&year=" + year)
          .then(res => {
            let result = res.data != null && res.data;
            resolve(result);
          })
          .catch(err => {
            this.response = err;
          });
      });
    },
    async createPeiCopy() {
      let question1Selection = this.getQuestionElement(1, 1);
      return await new Promise(resolve => {
        this.$ApiPost("/api/pei/createpeicopy", {
          PeiId: `${this.signedPeiId}`,
          ConfigAcademicYearId: question1Selection.selectedAnswerId
        })
          .then(res => {
            if (res && res.data != null) {
              resolve(res.data);

              //redirigir a detalle de PEI

              this.setPeiId(res.data.peiId);
              this.setLocationRejected(res.data.configLocationRejectFlag);
              this.$router.push({ path: "/pei/detail" });
            }
          })
          .catch(function(err) {
            this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
          });
      });
    },
     async createPSCopy() {
      this.ShowWait();
      let question1Selection = this.getQuestionElement(1,1);
      return await new Promise(resolve => {
        this.$ApiPost(
          "/api/pei/createpscopy", 
         { 
           PeiId: `${this.signedPeiId}`, 
           ConfigAcademicYearId: question1Selection.selectedAnswerId,
         }).then(res => {
            if (res && res.data != null) {
              resolve(res.data);
                            
              //redirigir a detalle de PEI              
              this.HideWait();
              this.setPeiId(res.data.peiId);
              this.setLocationRejected(res.data.configLocationRejectFlag);
              this.$router.push({ path: "/pei/detail" });
            }
          })
          .catch(function(err) {
            this.HideWait();
              Swal.mixin({
                  toast: true,
                  position: "top-end",
                  showConfirmButton: false,
                  timer: 6000
              }).fire(
                  err.message,
                  "Ha ocurrido un error mientras se procesaba la información.",
                  "error"
              );
          });
      });
    },
    async getRecommendedLocation(peiId) {
      return await new Promise(resolve => {
        this.$ApiGet("/api/pei/getrecommendedlocation?peiId=" + peiId)
          .then(res => {
            if (res) {
              resolve(res.data);
            }
          })
          .catch(err => {
            this.response = err;
          });
      });
    },
    async getAlternateLocations() {
      return await new Promise(resolve => {
        this.$ApiGet("/api/pei/getalternatelocations")
          .then(res => {
            if (res) {
              resolve(res.data);
            }
          })
          .catch(err => {
            this.response = err;
          });
      });
    },
    async updateStudentStatus(studentSie, status) {
      return await new Promise(resolve => {
        this.$ApiPut("/api/student/changestatus?studentsie=" + studentSie + "&status=" + status)
          .then(res => {
            if (res) {
              resolve(res.data);
            }
          })
          .catch(err => {
            this.showMessage(err, messageTypes.Error);
          });
      });
    },
    getStep(number) {
      return this.steps.find(step => step.stepNumber == number);
    },
    getStepElement(number) {
      let stepElement = this.$refs["questionGroup_" + number];
      if (stepElement != null) {
        return stepElement[0];
      }
      return null;
    },
    async isAgeValidForAcademicYear() {
      let studentId = this.$store.getters.SelectedStudent.studentId;
      return await new Promise(resolve => {
        this.$ApiGet("/api/student/isAgeValidForAcademicYear?studentSie=" + studentId + "&year=" + this.RealAcademicYear)
          .then(res => {
            if (res) {
              resolve(res.data);
            }
          })
          .catch(err => {
            this.response = err;
          });
      });
    },
    setQuestionVisible(stepNumber, questionNumber, isVisible) {
      let question = this.getQuestionElement(stepNumber, questionNumber);
      question.clear();
      question.isVisible = isVisible;
    },
    calculateStudentAge(dob) {
      let diffMs = Date.now() - dob.getTime();
      let ageDt = alert(new Date(diffMs));

      return Math.abs(ageDt.getUTCFullYear() - 1970);
    },
    getQuestionElement(stepNumber, questionNumber) {
      let stepElement = this.getStepElement(stepNumber);
      if (stepElement != null) {
        return stepElement.getQuestion(questionNumber);
      }
      return null;
    },
    getQuestionField(stepNumber, questionNumber, fieldId) {
      let questionElement = this.getQuestionElement(stepNumber, questionNumber);
      if (questionElement != null) {
        return questionElement.getField(fieldId);
      }
      return null;
    },
    async hasImpedimentYearBefore() {
      await this.GetLatestSignedPEI(this.RealAcademicYear - 1).then(result => {
        if (result.data != null) {
          let question6 = this.getQuestionElement(2, 6);
          question6.text = question6.text.replace("(anterior)", "<b>" + this.formatPeiNumber(result.data.peiId) + "</b>");
          if (result.data.configMobilityProblemFlag == 1) {
            this.setQuestionVisible(2, 6, true);
            let question6A = this.getQuestionElement(2, 6.1);
            question6A.text = "<p>¿Estas necesidades fueron expresadas en el PEI <b>" + this.formatPeiNumber(result.data.peiId) + "</b>?</p> ";
          }
        }
      });
    },
    async onQuestion1ValueChange(question) {
      if (question.selectedAnswer != undefined) {
        this.academicYear = question.selectedAnswerText ? question.selectedAnswerText : this.SelectedPeiYearText;
        this.RealAcademicYear = question.selectedAnswer;
        let yearId = question.selectedAnswerId ? question.selectedAnswerId : this.SelectedPeiYearId;
        this.revisionType = await this.getPeiType(this.studentSei, yearId);
        this.hasPeioPSTothisAcademciYearSelected = await this.hasSignedPei(this.studentSei, question.selectedAnswer);
        this.hasSignedPeiLastYear = await this.hasSignedPei(this.studentSei, this.RealAcademicYear - 1);
        this.resetWizard();
        let question1Element = this.getQuestionElement(1, 2);

        question1Element.text = "<p>¿Este PEI o PS se considera <b>" + this.revisionType + "</b> para el año <b>" + this.academicYear + "</b>?</p>";

        let pei = await this.getPeiPsNumberByYearSeiId(this.RealAcademicYear);

        if (pei != null) {
          let question2dot1Element = this.getQuestionElement(1, 2.1);
          this.signedPeiId = parseInt(pei.peiId);
          question2dot1Element.text =
            "<p>2(a) ¿Desea crear una copia de los datos ingresados para el PEI o PS  <b>" + this.formatPeiNumber(pei.peiId) + "<b/>?</p> ";
        }

        let question2Element = this.getQuestionElement(1, 2);
        let question3Element = this.getQuestionElement(1, 3);
        question2Element.clear();
        if (question3Element != null) {
          question3Element.clear();
        }
        this.setQuestionVisible(1, 2.1, false);
        this.hasImpedimentYearBefore();

        let questionRegions = this.getQuestionElement(4, 5.1);
        questionRegions.refreshOptions();
      }
    },
    async onQuestion2ValueChange(question) {
      //Opteniendo pregunta 1
      let question1Selection = this.getQuestionElement(1, 1);

      if (!question.selectedAnswer && question1Selection.selectedAnswer) {
        question.invalidSelection = true;
        question.validationResult.messageText = this.getValidationMessage("4");
      } else {
        question.invalidSelection = false;
      }
      this.validationForQuestion2a(question);
    },
    validationForQuestion2a(question) {
      if (question.selectedAnswer && this.RealAcademicYear != 0 && this.hasPeioPSTothisAcademciYearSelected) {
        this.setQuestionVisible(1, 2.1, true);
      } else {
        this.setQuestionVisible(1, 2.1, false);
      }
    },
    async hasAgencyLastYearSignedPei() {
      await this.GetLatestSignedPEI(this.RealAcademicYear - 1).then(result => {
        if (result.data) {
          
          //pregunta 2
          this.getQuestionElement(2, 2).text =
            "<p> En la Parte V del PEI <b>" +
            (this.RealAcademicYear - 2) +
            "-" +
            (this.RealAcademicYear - 1) +
            "</b> el estudiante tiene un programa de servicios" +
            " con actividades dirigidas al proceso de transición post-Secundario </p>";
          
          this.setQuestionVisible(2, 2, result.data.configTransitionServicesFlag == "1");
          //pregunta 3
          this.getQuestionElement(2, 3).text =
            "En la Parte IX del PEI <b>" +
            (this.RealAcademicYear - 2) +
            "-" +
            (this.RealAcademicYear - 1) +
            "</b> se estableció la necesidad de invitar a otras agencias para proveer algún servicio, ¿existe evidencia de que la agencia proveyó el servicio?";
          this.setQuestionVisible(2, 3, result.data.hasParticipantsAgencies);
        }
      });
    },
    isElegible() {
      let student = this.studentInfo;

      if (student.mipeStudent != null) {
        let studentStatus = student.mipeStudent.studentStatusCode;

        if (studentStatus == StudentStatuses.Elegible) return true;
        return false;
      }

      return false;
    },
    onQuestion2Step1ValueChange(question) {
      if (question.selectedAnswer != undefined && !question.selectedAnswer) {
        this.setQuestionVisible(2, 1.1, true);
      } else {
        this.setQuestionVisible(2, 1.1, false);
      }
    },
    onQuestion2Step2ValueChange(question) {
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        this.setQuestionVisible(2, 2.1, false);
      } else {
        this.setQuestionVisible(2, 2.1, true);
      }
    },
    onQuestion3ValueChange(question) {
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        this.setQuestionVisible(2, 3.1, false);
      } else {
        this.setQuestionVisible(2, 3.1, true);
      }
    },
    onQuestion3AValueChange() {},
    onQuestion4ValueChange(question) {
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        this.setQuestionVisible(2, 4.1, true);
        this.setQuestionVisible(2, 4.2, true);
      } else {
        let question4aElement = this.getQuestionElement(2, 4.1);
        let question4bElement = this.getQuestionElement(2, 4.2);
        let question4cElement = this.getQuestionElement(2, 4.3);

        question4aElement.clear();
        question4bElement.clear();
        question4cElement.clear();

        this.setQuestionVisible(2, 4.1, false);
        this.setQuestionVisible(2, 4.2, false);
        this.setQuestionVisible(2, 4.3, false);
      }
    },
    onQuestion4AValueChange(question) {
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        this.setQuestionVisible(2, 4.3, true);
      } else {
        let question4cElement = this.getQuestionElement(2, 4.3);
        question4cElement.clear();
        this.setQuestionVisible(2, 4.3, false);
      }
    },
    formatDateForPicker(date) {
      if (date != null) {
        date = date
          .toLocaleDateString("es-ES", {
            day: "numeric",
            month: "short",
            year: "numeric"
          })
          .replace(/ /g, "/");

        let firstSlash = date.indexOf("/");
        let formatted = date.substring(0, firstSlash) + "/" + date.charAt(firstSlash + 1).toUpperCase() + date.slice(firstSlash + 2);

        return formatted;
      }
      return null;
    },
    //evento provisional para evaluar cambio de status
    async CambioStatus() {
      let studentId = this.$store.getters.SelectedStudent.studentId;

      await this.updateStudentStatus(studentId, StudentStatuses.Servido);
    },
    onQuestion6ValueChange(question) {
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        this.setQuestionVisible(2, 6.1, true);
        this.setQuestionVisible(2, 6.2, true);
      } else {
        this.setQuestionVisible(2, 6.1, false);
        this.setQuestionVisible(2, 6.2, false);
        this.setQuestionVisible(2, 6.3, false);
        this.setQuestionVisible(2, 6.4, false);
        this.setQuestionVisible(2, 6.5, false);
        this.setQuestionVisible(2, 6.6, false);
      }
    },
    onQuestion6AValueChange(question) {
      let fieldComment = this.getQuestionElement(2, 6.1).getField("comentario");
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        fieldComment.visible = false;
        this.setQuestionVisible(2, 6.2, true);
      } else {
        fieldComment.visible = true;
        this.setQuestionVisible(2, 6.2, false);
      }
    },
    onQuestion6BValueChange(question) {
      let fieldComment = this.getQuestionElement(2, 6.2).getField("comentario");
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        fieldComment.visible = false;
        this.setQuestionVisible(2, 6.3, true);
      } else {
        fieldComment.visible = true;
        this.setQuestionVisible(2, 6.3, false);
      }
    },
    onQuestion6CValueChange(question) {
      let fieldComment = this.getQuestionElement(2, 6.3).getField("comentario");
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        this.setQuestionVisible(2, 6.4, true);
        this.setQuestionVisible(2, 6.5, true);
        this.setQuestionVisible(2, 6.6, true);
        fieldComment.visible = false;
      } else {
        this.setQuestionVisible(2, 6.4, false);
        this.setQuestionVisible(2, 6.5, false);
        this.setQuestionVisible(2, 6.6, false);
        fieldComment.visible = true;
      }
    },
    onQuestion6dot4FieldAjustesChange(event, field) {
      let fieldComment = this.getQuestionElement(2, 6.4).getField("comentario");
      fieldComment.visible = field.selectedOptions.includes("Otros.");
    },
    onQuestion6FValueChange(question) {
      let fieldComment = this.getQuestionElement(2, 6.6).getField("comentario");
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        fieldComment.visible = false;
      } else {
        fieldComment.visible = true;
      }
    },
    onQuestion7ValueChange(question) {
      if (question.selectedAnswer != undefined && !question.selectedAnswer) {
        question.invalidSelection = true;
        question.validationResult.messageText =
          "Pregunta 3: Usted no está de acuerdo con el impedimento \
                                 y no se puede proseguir. Favor de corregir el impedimento en MiPE para poder continuar.";
      } else {
        question.invalidSelection = false;
      }
    },
    async onQuestion8ValueChange(question) {
      if (question.selectedAnswer) {
        let isAgeValid = await this.isAgeValidForAcademicYear();
        if (!isAgeValid) {
          question.invalidSelection = true;
          question.validationResult.messageText =
            "Los estudiantes ubicados en un Head Start, \
                                                            Hogar o Centro de Cuido deben tener edad \
                                                            preescolar (2 años y 6 meses - 5 años). (41)";
        }
      } else {
        question.invalidSelection = false;
      }
    },
    async onQuestion9ValueChange(question) {
      if (question.selectedAnswer) {
        this.setQuestionVisible(3, 9.1, false);
        this.setQuestionVisible(3, 9.2, false);
        this.setQuestionVisible(3, 9.3, false);
        this.setQuestionVisible(3, 9.4, false);

        let question1Step1Element = this.getQuestionElement(1, 1);
        let studentEnrollment = await this.getStudentEnrollment(this.studentSei, question1Step1Element.selectedAnswerId);

        //si el estudiante no cuenta con matrí­cula activa
        if (studentEnrollment == null) {
          question.invalidSelection = true;
          question.validationResult.resultType = validationResultTypes.Warning;
          question.validationResult.messageText =
            "Pregunta 5: Su contestación no va de acuerdo con el SIE. \
                                                            Si este es un error, favor corregirlo en el SIE. ¿Está seguro(a) \
                                                            que desea continuar?. (10)";
        } else {
          question.invalidSelection = false;
        }
      } else {
        question.invalidSelection = false;
        this.setQuestionVisible(3, 9.1, true);
      }
      this.updateSchools();
    },
    onQuestion9dot1ValueChange(question) {
      let question8Answer = this.getQuestionElement(3, 8);
      if (question.selectedAnswer && question8Answer != undefined && !question8Answer.selectedAnswer) {
        question.invalidSelection = true;
        question.validationResult.messageText =
          "Si el estudiante estará ubicado en un Head Start o \
                                                      Centro de Cuido, debe contestar Sí a la pegunta #8. (40)";

        this.setQuestionVisible(3, 9.2, false);
        this.setQuestionVisible(3, 9.3, false);
        this.setQuestionVisible(3, 9.4, false);
      } else {
        if (question.selectedAnswer) {
          this.setQuestionVisible(3, 9.2, false);
        } else {
          this.setQuestionVisible(3, 9.2, true);
        }
        question.invalidSelection = false;
      }
      this.updateSchools();
    },
    async onQuestion9dot2ValueChange(question) {
      if (question.selectedAnswer) {
        let studentEnrollment = await this.getStudentEnrollment(this.studentSei, this.getQuestionElement(1, 1).selectedAnswerId);
        if (studentEnrollment) {
          question.validationResult.messageText = "El estudiante debe ser tipo Privado en el SIE. (22061)";
          question.invalidSelection = true;
        }
        this.setQuestionVisible(3, 9.3, false);
        this.setQuestionVisible(3, 9.4, false);
      } else {
        question.invalidSelection = false;
        this.setQuestionVisible(3, 9.3, true);
      }
      this.updateSchools();
    },
    onQuestion9dot3ValueChange(question) {
      if (question.selectedAnswer != undefined && !question.selectedAnswer) {
        question.invalidSelection = true;
        question.validationResult.messageText =
          "Pregunta 5.3: No puede crear un PEI o PS si el estudiante no está asignado \
                                                        a una escuela de DE y no es ubicación unilateral. \
                                                        Favor de revisar las contestaciones para poder continuar. (12)";

        this.setQuestionVisible(3, 9.4, false);
      } else {
        this.setQuestionVisible(3, 9.4, true);
        question.invalidSelection = false;
      }
      let peiConfigUnilateralFlag = question.selectedAnswer ? "1" : "0";
      this.$store.commit("setSelectedPeiConfigUnilateralFlag", peiConfigUnilateralFlag);
      this.updateSchools();
    },
    onQuestion9dot4ValueChange(question) {
      if (question.selectedAnswer != undefined && !question.selectedAnswer) {
        question.invalidSelection = true;
        question.validationResult.messageText = this.getValidationMessage("190902");
      } else {
        question.invalidSelection = false;
      }
      this.updateSchools();
    },
    async getPeiPsNumberByYearSeiId(academyYear) {
      let response = await this.GetLatestSignedPEI(academyYear);
      return response.data;
    },
    GetLatestSignedPEI(academyYear) {
      return new Promise((resolve, reject) => {
        this.$ApiGet("/api/pei/getlatestsignedpeips?studentSieId=" + this.studentSei + "&academicYear=" + academyYear).then(
          response => resolve(response),
          error => reject(error)
        );
      });
    },
    ClearQuestionSelectedAnswers(questions) {
      if (questions) {
        for (let i = 0; i < questions.length; i++) {
          questions[i].selectedAnswer = undefined;
          questions[i].questionOpenAnswer = "";
        }
      }
    },
    getRegions() {
      let promise = new Promise(async resolve => {
        let academicyear = !this.isFromPeiDetail
          ? this.getQuestionElement(1, 1).selectedAnswer
          : this.SelectedPeiYear
          ? this.SelectedPeiYear
          : this.getCurrentYear();
        if (academicyear != undefined) {
          let res = await this.$ApiGet("/api/pei/getregions?academicYear=" + academicyear);
          let datum = [];
          res.data.forEach(element => {
            datum.push({
              Value: element.id,
              Display: element.name,
              isSelected: false
            });
          });
          resolve(datum);
        }
      });
      return promise;
    },
    updateSchools() {
      let nextQuestion = this.getQuestionElement(4, 5.1);
      let isSchool = this.getQuestionElement(3, 9).selectedAnswer;
      let isLocation = this.getQuestionElement(3, 9.1).selectedAnswer;
      let fieldSchoold = nextQuestion.getField("escuela");

      if (isSchool != undefined && isSchool) {
        fieldSchoold.label = "<b>Escuela:</b>";
      } else if (isLocation != undefined && isLocation) {
        fieldSchoold.label = "<b>Tipo de Ubicación:</b>";
      } else {
        fieldSchoold.label = "<b>Instituciones:</b>";
      }

      //nextQuestion.refreshOptions();

      let fieldRegion = nextQuestion.getField("region");
      fieldRegion.value = null;

      let fieldDistrict = nextQuestion.getField("distrito");
      fieldDistrict.value = null;
      fieldDistrict.options = [];

      fieldSchoold.value = null;
      fieldSchoold.options = [];
    },
    getDistricts(academicYear, regionId) {
      let promise = new Promise(async resolve => {
        if (academicYear != undefined && regionId != null) {
          let res = await this.$ApiGet("/api/pei/getdistricts?regionId=" + regionId + "&academicYear=" + academicYear);
          let datum = [];
          res.data.forEach(element => {
            datum.push({
              Value: element.id,
              Display: element.name,
              isSelected: false
            });
          });
          resolve(datum);
        }
      });
      return promise;
    },
    getGrados() {
      let promise = new Promise(async resolve => {
        let res = await this.$ApiGet("/api/pei/getgrades");
        let datum = [];
        res.data.forEach(element => {
          datum.push({
            Value: element.id,
            Display: element.name,
            isSelected: false
          });
        });
        resolve(datum);
      });
      return promise;
    },
    onQuestion6Step4ValueChange(question) {
      if (question.selectedAnswer != undefined) {
        let gradoActual = this.studentInfo.mipeStudent.gradeId;
        let res = gradoActual + 1 === question.selectedAnswer || gradoActual === question.selectedAnswer;
        if (!res) {
          question.invalidSelection = true;
          question.validationResult.resultType = validationResultTypes.Warning;
          question.validationResult.messageText = "El grado seleccionado no es el mismo o el próximo segun los datos en el sistema.";
        } else {
          question.invalidSelection = false;
        }
      }
    },
    getSchools(academicYear, district, isUnilateral, isPublic, isSpecialSchool, isInstitution) {
      let promise = new Promise(async resolve => {
        let SchoolTypesEnum = {
          CareCenter: 246,
          PrivateSchool: 248,
          PublicSchool: 249,
          HeadStart: 250,
          Home: 251,
          Institutions: 349
        };

        let schoolTypes = [];

        if (isSpecialSchool) {
          schoolTypes.push(SchoolTypesEnum.CareCenter);
          schoolTypes.push(SchoolTypesEnum.HeadStart);
          schoolTypes.push(SchoolTypesEnum.Home);
        }
        if (isPublic != undefined && isPublic) {
          schoolTypes.push(SchoolTypesEnum.PublicSchool);
        }
        if (isInstitution) {
          schoolTypes.push(SchoolTypesEnum.Institutions);
        } else if (isUnilateral == "1") {
          schoolTypes.push(SchoolTypesEnum.PrivateSchool);
        }

        if (district != undefined) {
          let academicYearIdRequest = academicYear === undefined ? '' : `academicYear=${academicYear}&`;
          let schoolTypesParam = schoolTypes.length > 0 ? `&schoolTypes=${schoolTypes.join("&schoolTypes=")}` : '';
          let requestUrl = `/api/schools/?${academicYearIdRequest}districtId=${district}${schoolTypesParam}`;
          let res = await this.$ApiGet(requestUrl);
          let datum = [];
          res.data.forEach(element => {
            datum.push({
              Value: element.id,
              Display: element.name,
              isSelected: false
            });
          });
          resolve(datum);
        }
      });
      return promise;
    },
    onQuestion8Step4ValueChange(question) {
      let nextQuestion = this.getQuestionElement(4, 8.1);
      if (question.selectedAnswer != undefined && question.selectedAnswer) {
        nextQuestion.isVisible = true;
        nextQuestion.isRequired = true;
        nextQuestion.refreshOptions();
      } else {
        nextQuestion.isVisible = false;
        nextQuestion.isRequired = false;
      }
    },
    getTransitions() {
      let promise = new Promise(async resolve => {
        let studentAge = this.$store.getters.SelectedStudent.age;

        let isPreSchool = studentAge >= peiTransitionAges.TwoAndSevenMonth && studentAge <= peiTransitionAges.Six;

        let res = await this.$ApiGet("/api/pei/gettransitions?isPreSchool=" + isPreSchool);
        let datum = [];
        res.data.forEach(element => {
          datum.push({
            Value: element.id,
            Display: element.name,
            isSelected: false
          });
        });
        resolve(datum);
      });
      return promise;
    },
    getMeasurePrograms() {
      let measurePrograms = [
        { value: 16, option: "Regular" },
        { value: 17, option: "Evaluación Alterna" },
        { value: 18, option: "No Aplica" }
      ];
      return this.resolveOptions(measurePrograms);
    },
    onQuestion11Step4ValueChange(question) {
      let nextQuestion = this.getQuestionElement(4, 11.1);
      if (question.selectedAnswer != undefined && question.selectedAnswer === 16) {
        nextQuestion.isVisible = true;
        nextQuestion.isRequired = true;
      } else {
        nextQuestion.isVisible = false;
        nextQuestion.isRequired = false;
      }
    },
    formatPeiNumber(pei) {
      let formatted = pei.substring(0, 4) + "-" + pei.substring(4, pei.length);
      return formatted;
    },
    async getStudentEnrollment(studentSie, academicYearId) {
      let res = await this.$ApiGet("/api/student/getenrollment?studentSie=" + studentSie + "&academicYearId=" + academicYearId);
      return res.data;
    },
    resetWizard() {
      this.isCurrentStepValid = true;
      for (let stepIdx = 0; stepIdx < this.steps.length; stepIdx++) {
        const step = this.steps[stepIdx];
        if (step.questions) {
          for (let questionIdx = 0; questionIdx < step.questions.length; questionIdx++) {
            const question = step.questions[questionIdx];
            if (step.stepNumber != 1) {
              this.getQuestionElement(step.stepNumber, question.number).clear();
              this.getQuestionElement(step.stepNumber, question.number).isVisible = question.initialState;
            }
          }
        }
      }
    },
    parseFlag(value) {
      if (value === "1") {
        return 1
      }

      if (value === "0") {
        return 0
      }

      return null
    },
    async mapPSAnswersToQuestions(){
      let pei = this.PeiGeneralDB;

      //paso año académico
      this.getQuestionElement(1, 1).selectedAnswer = this.SelectedPeiYear;
      this.getQuestionElement(1, 1).disabled = true;

      //Como se está actualizando datos de define PEI, esta pregunta es una validación operacional y no se guarda en la base de datos, por lo que siempre será Si en modo readOnly, porque el PEI ya está definido.
      this.getQuestionElement(1, 2).selectedAnswer = 1;
      this.getQuestionElement(1, 2).disabled = true;

      this.getQuestionElement(1, 2.1).isVisible = false;
      //paso detalle PS
      this.getQuestionElement(4, 6).selectedAnswer = pei.gradeId;
      this.getQuestionElement(4, 7).selectedAnswer = parseInt(pei.inapropriateConductFlag);
      this.getQuestionElement(4, 7.1).selectedAnswer = parseInt(pei.unilateralServiceRelatedFlag);

      this.getQuestionElement(3, 7).selectedAnswer = parseInt(pei.impedimentFlag);
      this.getQuestionElement(3, 7).disabled = true;

      this.getQuestionElement(3, 8).selectedAnswer = parseInt(pei.prescholarFlag);
      this.getQuestionElement(3, 8).disabled = true;
      
      this.getQuestionElement(3, 9).selectedAnswer = parseInt(pei.publicSchoolFlag);
      if(this.getQuestionElement(3, 9).selectedAnswer == 0){
        await this.onQuestion9ValueChange(this.getQuestionElement(3, 9));

        this.getQuestionElement(3, 9).disabled = true;

        this.getQuestionElement(3, 9.1).selectedAnswer = parseInt(pei.peiRegularSpecialFlag);
        await this.onQuestion9dot1ValueChange(this.getQuestionElement(3, 9.1));
        this.getQuestionElement(3, 9.1).disabled = true;

        this.getQuestionElement(3, 9.2).selectedAnswer = parseInt(pei.configInstitutionalFlag);
        await this.onQuestion9dot2ValueChange(this.getQuestionElement(3, 9.2));
        this.getQuestionElement(3, 9.2).disabled = true;

        this.getQuestionElement(3, 9.3).selectedAnswer = parseInt(pei.unilateralFlag);
        await this.onQuestion9dot3ValueChange(this.getQuestionElement(3, 9.3));
        this.getQuestionElement(3, 9.3).disabled = true;

        let question9dot4Element = this.getQuestionElement(3, 9.4);
        question9dot4Element.selectedAnswer = parseInt(pei.configLocationRejectFlag);

        if(pei.configLocationRejectFlag == 1 && pei.locationRejectDocument != null){
          question9dot4Element.getField("fecha").value = pei.locationRejectDocument.documentDate;
          let attachmentField = question9dot4Element.getField("attachment");
          attachmentField.attachmentId = pei.locationRejectDocument.itemAttachmentId;
        }

        await this.onQuestion9dot4ValueChange(this.getQuestionElement(3, 9.4));
        this.getQuestionElement(3, 9.4).disabled = true;

      }

      //location
      let selectedRegion = null;
      await this.getRegions()
        .then(res => {
          selectedRegion = res.find(x => x.Value == pei.locationRegionId);
          this.getQuestionField(4, 5.1, "region").value = selectedRegion;
        })
        .catch(err => {
          this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
        });

      if (selectedRegion != null) {
        await this.onRegionChange(selectedRegion);
        let selectedDistric = null;
        await this.getDistricts(this.SelectedPeiYear, selectedRegion.Value)
          .then(res => {
            selectedDistric = res.find(x => x.Value == pei.configLocationDistrictId);
            this.getQuestionField(4, 5.1, "distrito").value = selectedDistric;
          })
          .catch(err => {
            this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
          });

        if (selectedDistric != null) {
          await this.onDistrictChange(selectedDistric);
          let isPublic = this.getQuestionElement(3, 9).selectedAnswer;
          let isSpecialSchool = this.getQuestionElement(3, 9.1).selectedAnswer;
          let isInstitution = this.getQuestionElement(3, 9.2).selectedAnswer;
          let isUnilateral = this.getQuestionElement(3, 9.3).selectedAnswer;
          let selectedAnswerYearId = this.getQuestionElement(1, 1).selectedAnswerId;

          let selectedSchool = null;
          await this.getSchools(selectedAnswerYearId, selectedDistric.Value, isUnilateral, isPublic, isSpecialSchool, isInstitution)
            .then(res => {
              selectedSchool = res.find(x => x.Value == pei.peiLocationId);
              this.getQuestionField(4, 5.1, "escuela").value = selectedSchool;
            })
            .catch(err => {
              this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
            });
        }
      }

    },
    async mapPeiAnswersToQuestions() {
      let pei = this.PeiGeneralDB;

      //paso año académico
      this.getQuestionElement(1, 1).selectedAnswer = this.SelectedPeiYear;
      await this.onQuestion1ValueChange(this.getQuestionElement(1, 1));
      this.getQuestionElement(1, 1).disabled = true;

      //Como se está actualizando datos de define PEI, esta pregunta es una validación operacional y no se guarda en la base de datos, por lo que siempre será Si en modo readOnly, porque el PEI ya está definido.
      this.getQuestionElement(1, 2).selectedAnswer = 1;
      this.getQuestionElement(1, 2).disabled = true;

      this.getQuestionElement(1, 2.1).isVisible = false;

      //paso preguntas adicionales---------------------------------------------------------------------------------------------------
      this.getStep(2).showStep = await this.hasSignedPeiLastYear;

      //alternativa de ubicacion
      let questionUbication = pei.peiAdditionalQuestions.find(x => x.peiAdditionalQuestionId == PeiAdditionalQuestionType.Evidencia_de_Participación_de_Alternatia_de_Ubicación);
      if (questionUbication != undefined) {
        this.getQuestionElement(2, 1).selectedAnswer = questionUbication.peiAdditionalQuestionAnswerId == EduEspYesNoNA.Yes ? 1 : 0;
        await this.onQuestion2Step1ValueChange(this.getQuestionElement(2, 1));
        this.getQuestionElement(2, 1).disabled = true;
        if (!this.getQuestionElement(2, 1).selectedAnswer) {
          this.getQuestionElement(2, 1.1).selectedAnswer = questionUbication.alternatePlacementId;
          this.getQuestionElement(2, 1.1).disabled = true;
        }
      }

      //Validate for question P2 and P3
      await this.hasAgencyLastYearSignedPei();
      //post-secundario
      let questionPostSecundary = pei.peiAdditionalQuestions.find(x => x.peiAdditionalQuestionId == PeiAdditionalQuestionType.Evidencia_de_Participación_Servicio_Transición_Post_Secundario);
      if (questionPostSecundary != undefined) {
        if (questionPostSecundary.peiAdditionalQuestionAnswerId) {
          this.getQuestionElement(2, 2).selectedAnswer = questionPostSecundary.peiAdditionalQuestionAnswerId == EduEspYesNoNA.Yes ? 1 : 0;
          await this.onQuestion2Step2ValueChange(this.getQuestionElement(2, 2));
          this.getQuestionElement(2, 2).disabled = true;
          if (!this.getQuestionElement(2, 2).selectedAnswer) {
            this.getQuestionElement(2, 2.1).questionOpenAnswer = questionPostSecundary.additionalQuestionComment;
            this.getQuestionElement(2, 2.1).disabled = true;
          }
        } else { this.setQuestionVisible(2,2, false); }
      }

      //Evidencia agencia
      let questionAgency = pei.peiAdditionalQuestions.find(x => x.peiAdditionalQuestionId == PeiAdditionalQuestionType.Evidencia_de_que_la_Agencia_Proveyó_el_Servicio);
      if (questionAgency != undefined) {
        if (questionAgency.peiAdditionalQuestionAnswerId) {
          this.getQuestionElement(2, 3).selectedAnswer = questionAgency.peiAdditionalQuestionAnswerId == EduEspYesNoNA.Yes ? 1 : 0;
          await this.onQuestion3ValueChange(this.getQuestionElement(2, 3));
          this.getQuestionElement(2, 3).disabled = true;          
          if (questionAgency.peiAdditionalQuestionAnswerId == EduEspYesNoNA.No && questionAgency.additionalQuestionComment.length > 0) {
            this.getQuestionElement(2, 3.1).questionOpenAnswer = questionAgency.additionalQuestionComment;
            this.getQuestionElement(2, 3.1).disabled = true;
          }else {
            this.setQuestionVisible(2, 3.1, false);
          }
        } else { this.setQuestionVisible(2, 3, false); }
      }
        
      //asistente de servicio
      let questionService = pei.peiAdditionalQuestions.find(x => x.peiAdditionalQuestionId == PeiAdditionalQuestionType.Asistente_de_Servicio);
      if (questionService != undefined) {
        this.getQuestionElement(2, 4).selectedAnswer = questionService.peiAdditionalQuestionAnswerId == EduEspYesNoNA.Yes ? 1 : 0;
        await this.onQuestion4ValueChange(this.getQuestionElement(2, 4));
        this.getQuestionElement(2, 4).disabled = true;
        if (this.getQuestionElement(2, 4).selectedAnswer) {
          this.getQuestionElement(2, 4.2).selectedAnswer = questionService.serviceAssistantServiceId;
          this.getQuestionElement(2, 4.2).disabled = true;
        }
      }

      let questionUniqueService = pei.peiAdditionalQuestions.find(x => x.peiAdditionalQuestionId == PeiAdditionalQuestionType.Asistente_de_Servicio_Único);
      if (questionUniqueService != undefined) {
        this.getQuestionElement(2, 4.1).selectedAnswer = questionUniqueService.peiAdditionalQuestionAnswerId == EduEspYesNoNA.Yes ? 1 : 0;
        await this.onQuestion4AValueChange(this.getQuestionElement(2, 4.1));
        this.getQuestionElement(2, 4.1).disabled = true;
        if (this.getQuestionElement(2, 4.1).selectedAnswer) {
          this.getQuestionElement(2, 4.3).questionOpenAnswer = questionUniqueService.serviceAssistantName;
          this.getQuestionElement(2, 4.3).disabled = true;
        }
      }

      //fecha ubicacion
      let questionUbicationDate = pei.peiAdditionalQuestions.find(x => x.peiAdditionalQuestionId == PeiAdditionalQuestionType.Fecha_de_Ubicación);
      if (questionUbicationDate != undefined) {
        this.getQuestionElement(2, 5).isVisible = true;
        this.getQuestionElement(2, 5).selectedAnswer = questionUbicationDate.alternatePlacementDate;
        this.getQuestionElement(2, 5).disabled = true;
      }

      this.onQuestion2Step1ValueChange(this.getQuestionElement(2, 1));
      this.getQuestionElement(2, 1).disabled = true;

      //paso Inicio
      this.getQuestionElement(3, 7).selectedAnswer = this.parseFlag(pei.impedimentFlag);
      this.getQuestionElement(3, 7).disabled = true;

      this.getQuestionElement(3, 8).selectedAnswer = this.parseFlag(pei.prescholarFlag);
      this.getQuestionElement(3, 8).disabled = true;

      this.getQuestionElement(3, 9).selectedAnswer = this.parseFlag(pei.publicSchoolFlag);
      this.getQuestionElement(3, 9).disabled = true;
      if(this.getQuestionElement(3, 9).selectedAnswer == 0){
        await this.onQuestion9ValueChange(this.getQuestionElement(3, 9));

        this.getQuestionElement(3, 9).disabled = true;

        this.getQuestionElement(3, 9.1).selectedAnswer = this.parseFlag(pei.peiRegularSpecialFlag);
        await this.onQuestion9dot1ValueChange(this.getQuestionElement(3, 9.1));
        this.getQuestionElement(3, 9.1).disabled = true;

        this.getQuestionElement(3, 9.2).selectedAnswer = this.parseFlag(pei.configInstitutionalFlag);
        await this.onQuestion9dot2ValueChange(this.getQuestionElement(3, 9.2));
        this.getQuestionElement(3, 9.2).disabled = true;

        this.getQuestionElement(3, 9.3).selectedAnswer = this.parseFlag(pei.unilateralFlag);
        await this.onQuestion9dot3ValueChange(this.getQuestionElement(3, 9.3));
        this.getQuestionElement(3, 9.3).disabled = true;

        this.getQuestionElement(3, 9.4).selectedAnswer = this.parseFlag(pei.configLocationRejectFlag);
        this.getQuestionElement(3, 9.4).disabled = true;
      }
      //paso detalle (No disabled)
      this.getQuestionElement(4, 6).selectedAnswer = pei.gradeId;
      this.getQuestionElement(4, 7).selectedAnswer = parseInt(pei.inapropriateConductFlag);
      this.getQuestionElement(4, 8).selectedAnswer = parseInt(pei.transitionServicesFlag);
      
      if (pei.unilateralFlag == "0") {
        if (this.getQuestionElement(4,8).selectedAnswer == 1)
          await this.onQuestion8Step4ValueChange(this.getQuestionElement(4,8));
  
        this.getQuestionElement(4, 8.1).selectedAnswer = pei.peiTransitionCode;
        this.getQuestionElement(4, 9).selectedAnswer = parseInt(pei.impedimentCognativeFlag);
        this.getQuestionElement(4, 10).selectedAnswer = parseInt(pei.placementFlag);
        this.getQuestionElement(4, 11).selectedAnswer = pei.peiOtherServMeasureCode;
        this.getQuestionElement(4, 12).selectedAnswer = parseInt(pei.otherServTechAssistEquipFlag);
        this.getQuestionElement(4, 13).selectedAnswer = parseInt(pei.extendedSchoolYearFlag);
        this.getQuestionElement(4, 14).selectedAnswer = parseInt(pei.configMobilityProblemFlag);
        this.getQuestionElement(4, 15).selectedAnswer = parseInt(pei.configSignsLanguageFlag);
        this.getQuestionElement(4, 16).selectedAnswer = parseInt(pei.configBrailleFlag);
  
        if (pei.peiOtherServMeasureCode == 16) {
          await this.onQuestion11Step4ValueChange(this.getQuestionElement(4, 11));
          this.getQuestionElement(4, 11.1).selectedAnswer = parseInt(pei.otherServMeasurePlacementFlag);
        }
      }

      //location
      let selectedRegion = null;
      await this.getRegions()
        .then(res => {
          selectedRegion = res.find(x => x.Value == pei.regionId);
          if (pei.publicSchoolFlag == "0") {
            selectedRegion = res.find(x => x.Value == pei.locationRegionId);
          }
          this.getQuestionField(4, 5.1, "region").value = selectedRegion;
        })
        .catch(err => {
          this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
        });

      if (selectedRegion != null) {
        await this.onRegionChange(selectedRegion);
        let selectedDistric = null;
        await this.getDistricts(this.SelectedPeiYear, selectedRegion.Value)
          .then(res => {
            selectedDistric = res.find(x => x.Value == pei.districtId);
            if (pei.publicSchoolFlag == "0") {
              selectedDistric = res.find(x => x.Value == pei.configLocationDistrictId);
            }
            this.getQuestionField(4, 5.1, "distrito").value = selectedDistric;
          })
          .catch(err => {
            this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
          });

        if (selectedDistric != null) {
          await this.onDistrictChange(selectedDistric);
          let isPublic = this.getQuestionElement(3, 9).selectedAnswer;
          let isSpecialSchool = this.getQuestionElement(3, 9.1).selectedAnswer;
          let isInstitution = this.getQuestionElement(3, 9.2).selectedAnswer;
          let isUnilateral = this.getQuestionElement(3, 9.3).selectedAnswer;
          let selectedAnswerYearId = this.getQuestionElement(1, 1).selectedAnswerId;

          let selectedSchool = {};
          await this.getSchools(selectedAnswerYearId, selectedDistric.Value, isUnilateral, isPublic, isSpecialSchool, isInstitution)
            .then(res => {
              selectedSchool = res.find(x => x.Value == pei.schoolId);
              if (pei.publicSchoolFlag == "0") {
                selectedSchool = res.find(x => x.Value == pei.peiLocationId);
              }
              this.getQuestionField(4, 5.1, "escuela").value = selectedSchool;
            })
            .catch(err => {
              this.ShowToast(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
            });
        }
      }
    },
    mapQuestionsAnswers() {
      let pei = this.PeiGeneral;
      pei.studentSieId = parseInt(this.studentSei);

      try {
        pei.ConfigAcademicYearId = this.getQuestionElement(1, 1).selectedAnswer;
        pei.IsRevision = this.getQuestionElement(1, 2).selectedAnswer ? 1 : 0;

        if (pei.IsRevision == 1) {
          pei.PeiTypeCode = peiTypes.Revision;
        } else {
          pei.PeiTypeCode = peiTypes.Initial;
        }

        pei.ConfigAgeAmt = this.$store.getters.SelectedStudent.age;
        //Unilateral siempre será false al crear PEI desde el wizard
        pei.IsUnilateral = 0; // this.getQuestionElement(3, 9.3).selectedAnswer ? 1 : 0; 

        //paso preguntas adicionales---------------------------------------------------------------------------------------------------
        //alternativa de ubicacion
        let question1 = this.getQuestionElement(2, 1);
        let paPlacement = this.PeiAdditionalQuestion;
        paPlacement.PeiAdditionalQuestionId = PeiAdditionalQuestionType.Evidencia_de_Participación_de_Alternatia_de_Ubicación;
        paPlacement.PeiAdditionalQuestionAnswerId = question1.selectedAnswer ? EduEspYesNoNA.Yes : EduEspYesNoNA.No;

        if (!question1.selectedAnswer) {
          paPlacement.AlternatePlacementId = this.getQuestionElement(2, 1.1).selectedAnswer;
        }

        pei.PeiAdditionalQuestions.push(paPlacement);

        //post-secundario
        let question2 = this.getQuestionElement(2, 2);
        this.PeiAdditionalQuestion = {};
        let paPostSecundario = this.PeiAdditionalQuestion;
        paPostSecundario.PeiAdditionalQuestionId = PeiAdditionalQuestionType.Evidencia_de_Participación_Servicio_Transición_Post_Secundario;
        paPostSecundario.PeiAdditionalQuestionAnswerId = question2.selectedAnswer ? EduEspYesNoNA.Yes : EduEspYesNoNA.No;

        if (!question2.selectedAnswer) {
          paPostSecundario.AdditionalQuestionComment = this.getQuestionElement(2, 2.1).questionOpenAnswer;
        }

        pei.PeiAdditionalQuestions.push(paPostSecundario);

        //agencia
        let question3 = this.getQuestionElement(2, 3);
        this.PeiAdditionalQuestion = {};
        let paAgencyParticipation = this.PeiAdditionalQuestion;
        paAgencyParticipation.PeiAdditionalQuestionId = PeiAdditionalQuestionType.Evidencia_de_que_la_Agencia_Proveyó_el_Servicio;
        paAgencyParticipation.PeiAdditionalQuestionAnswerId = question3.selectedAnswer ? EduEspYesNoNA.Yes : EduEspYesNoNA.No;

        if (!question3.selectedAnswer) {
          paAgencyParticipation.AdditionalQuestionComment = this.getQuestionElement(2, 3.1).questionOpenAnswer;
        }

        pei.PeiAdditionalQuestions.push(paAgencyParticipation);

        //asistente de servicio
        let question4 = this.getQuestionElement(2, 4);
        this.PeiAdditionalQuestion = {};
        let paServiceAssistant = this.PeiAdditionalQuestion;
        paServiceAssistant.PeiAdditionalQuestionId = PeiAdditionalQuestionType.Asistente_de_Servicio;
        paServiceAssistant.PeiAdditionalQuestionAnswerId = question4.selectedAnswer ? EduEspYesNoNA.Yes : EduEspYesNoNA.No;

        if (question4.selectedAnswer) {
          paServiceAssistant.ServiceAssistantServiceId = this.getQuestionElement(2, 4.2).selectedAnswer;
        }

        pei.PeiAdditionalQuestions.push(paServiceAssistant);

        let question4a = this.getQuestionElement(2, 4.1);
        this.PeiAdditionalQuestion = {};
        let paServiceAssistantSingleStudent = this.PeiAdditionalQuestion;
        paServiceAssistantSingleStudent.PeiAdditionalQuestionId = PeiAdditionalQuestionType.Asistente_de_Servicio_Único;
        paServiceAssistantSingleStudent.PeiAdditionalQuestionAnswerId = question4a.selectedAnswer ? EduEspYesNoNA.Yes : EduEspYesNoNA.No;

        if (question4a.selectedAnswer) {
          paServiceAssistantSingleStudent.ServiceAssistantName = this.getQuestionElement(2, 4.3).questionOpenAnswer;
        }

        pei.PeiAdditionalQuestions.push(paServiceAssistantSingleStudent);

        //fecha ubicacion
        let question5 = this.getQuestionElement(2, 5);
        if (question5.selectedAnswer) {
          this.PeiAdditionalQuestion = {};
          let paAlternatePlacementDate = this.PeiAdditionalQuestion;
          paAlternatePlacementDate.PeiAdditionalQuestionId = PeiAdditionalQuestionType.Fecha_de_Ubicación;
          paAlternatePlacementDate.PeiAdditionalQuestionAnswerId = question4a.selectedAnswer ? EduEspYesNoNA.Yes : EduEspYesNoNA.No;
          paAlternatePlacementDate.AlternatePlacementDate = question5.selectedAnswer;
          pei.AlternatePlacementDate = paAlternatePlacementDate.AlternatePlacementDate;
          pei.PeiAdditionalQuestions.push(paAlternatePlacementDate);
        }

        //FIN paso preguntas adicionales-------------------------------------------------------------------------------------------------
        //paso Inicio
        pei.ConfigImpedimentFlag = this.getQuestionElement(3, 7).selectedAnswer ? 1 : 0;
        pei.ConfigPrescholarFlag = this.getQuestionElement(3, 8).selectedAnswer ? 1 : 0;
        pei.ConfigAgeAmtFlag = pei.ConfigPrescholarFlag;
        pei.ConfigPublicSchoolFlag = this.getQuestionElement(3, 9).selectedAnswer ? 1 : 0;
        pei.IsSpecialPrivateStudent = this.getQuestionElement(3, 9.1).selectedAnswer ? 1 : 0;
        pei.IsInstitutional = this.getQuestionElement(3, 9.2).selectedAnswer ? 1 : 0;
        //Unilateral siempre será false al crear PEI desde el wizard
        pei.ConfigUnilateralFlag = 0; // this.getQuestionElement(3, 9.3).selectedAnswer ? 1 : 0; 
        
        pei.ConfigLocationRejectFlag = this.getQuestionElement(3,9.4).selectedAnswer ? 1 : 0;
        //paso detalle
        let question6step4 = this.getQuestionElement(4, 6);
        pei.ConfigGradeId = question6step4.selectedAnswer;
        pei.ConfigInapropriateConductFlag = this.getQuestionElement(4, 7).selectedAnswer ? 1 : 0;
        pei.ConfigTransitionServicesFlag = this.getQuestionElement(4, 8).selectedAnswer ? 1 : 0;
        pei.PeiTransitionCode = this.getQuestionElement(4, 8.1).selectedAnswer;
        pei.ConfigImpedimentCognativeFlag = this.getQuestionElement(4, 9).selectedAnswer ? 1 : 0;
        pei.ConfigPlacementFlag = this.getQuestionElement(4, 10).selectedAnswer ? 1 : 0;
        pei.PeiOtherServMeasureCode = this.getQuestionElement(4, 11).selectedAnswer;
        pei.ConfigOtherServTechAssistEquipFlag = this.getQuestionElement(4, 12).selectedAnswer ? 1 : 0;
        pei.ConfigExtendedSchoolYearFlag = this.getQuestionElement(4, 13).selectedAnswer ? 1 : 0;
        pei.ConfigMobilityProblemFlag = this.getQuestionElement(4, 14).selectedAnswer ? 1 : 0;
        pei.ConfigSignsLanguageFlag = this.getQuestionElement(4, 15).selectedAnswer ? 1 : 0;
        pei.ConfigBrailleFlag = this.getQuestionElement(4, 16).selectedAnswer ? 1 : 0;

        

        if (pei.ConfigLocationRejectFlag) {
          //ubicación
          pei.ConfigLocationRegionId = null;
          pei.ConfigLocationDistrictId = null;
          pei.PeiLocationId = null;

          pei.ConfigOtherServMeasurePlacementFlag = 0;

          //Cuando se está actualizando el ps
          if(this.$store.state.pei.configUnilateralFlag){
            //cuando se crean pei cortos ubicacion es null
            pei.RegionId = this.getQuestionField(4, 5.1, "region").value ? this.getQuestionField(4, 5.1, "region").value.Value : null;
            pei.DistrictId = this.getQuestionField(4, 5.1, "region").value ? this.getQuestionField(4, 5.1, "distrito").value.Value : null;
            pei.ConfigSchoolId = this.getQuestionField(4, 5.1, "region").value ? this.getQuestionField(4, 5.1, "escuela").value.Value : null;
          }
        } else {          
          //ubicación
          if (pei.IsInstitutional) {
            pei.ConfigLocationRegionId = this.getQuestionField(4, 5.1, "region").value.Value;
            pei.ConfigLocationDistrictId = this.getQuestionField(4, 5.1, "distrito").value.Value;
            pei.GeneralStudentPublicSchoolFlag = 0;
          } else {
            pei.RegionId = this.getQuestionField(4, 5.1, "region").value.Value;
            pei.DistrictId = this.getQuestionField(4, 5.1, "distrito").value.Value;
            pei.GeneralStudentPublicSchoolFlag = 1;
          }

          if (pei.IsSpecialPrivateStudent) {
            pei.ConfigSchoolId = this.getQuestionField(4, 5.1, "escuela").value.Value;
          } else if (pei.IsInstitutional) {
            pei.PeiLocationId = this.getQuestionField(4, 5.1, "escuela").value.Value;
          } else {
            pei.ConfigSchoolId = this.getQuestionField(4, 5.1, "escuela").value.Value;
          }
        }

        if(pei.configUnilateralFlag){
          pei.ConfigInapropriateConductFlag = 0;
          pei.ConfigUnilateralServiceEducationalFlag = 0;
          pei.ConfigUnilateralServiceRelatedFlag = 1;
          pei.ConfigTransitionServicesFlag = 0;
          pei.PeiTransitionCode = null;
          pei.PeiOtherServMeasureCode = null;
          pei.ConfigOtherServMeasurePlacementFlag = 0;
          pei.ConfigImpedimentCognativeFlag = 0;
          pei.ConfigPlacementFlag = 0;
          pei.ConfigOtherServTechAssistEquipFlag = 0;
          pei.ConfigExtendedSchoolYearFlag = 0;
          pei.ConfigPublicSchoolFlag = 0;
          pei.GeneralStudentPublicSchoolFlag = 0;
        }else{
          pei.PeiUnilateralLocationId = null;
          pei.ConfigUnilateralServiceEducationalFlag = 0;
          pei.ConfigUnilateralServiceRelatedFlag = 0;

          if (!pei.ConfigTransitionServicesFlag) {
            pei.PeiTransitionCode = null;
          }

          if (pei.PeiOtherServMeasureCode != 16) {
            pei.ConfigOtherServMeasurePlacementFlag = 0;
          } else {
            pei.ConfigOtherServMeasurePlacementFlag = this.getQuestionElement(4, 11.1).selectedAnswer ? 1 : 0;
          }
        }

        //si estamos completando un PS se asigna lo respondido en la pregunta
        if (this.isFromPeiDetail) {
          pei.ConfigUnilateralServiceRelatedFlag = this.getQuestionElement(4, 7.1).selectedAnswer ? 1 : 0;
        }
        this.PeiGeneral = pei;
      } catch (error) {
        Swal.mixin({
          toast: true,
          position: "top-end",
          showConfirmButton: false,
          timer: 6000
        }).fire(error, "Ha ocurrido un error mientras se procesaba la información.", "error");
      }
    },
    async createPei() {
      
      try {

          let res = await this.$ApiPost("/api/pei/create", this.PeiGeneral);

          if (res && res.data != null) {
            
            await this.uploadRejectedLocationDocument(res.data.peiId);
            
            //cambiar status estudiante a Servido
            if (this.PeiGeneral.AlternatePlacementDate) {
              this.CambioStatus();
            }

              //redirigir a detalle de PEI
              this.$store.commit("setSelectedPeiConfigUnilateralFlag", res.data.unilateralFlag);
              this.setPeiId(res.data.peiId);
              this.setLocationRejected(res.data.configLocationRejectFlag);
         
         
              //Solo Pei Regular y PS van a poder acceder a enmienda
              if(res.data.generalRevisionNo > 0 && ((res.data.unilateralFlag == "0" && res.data.configLocationRejectFlag == "0") ||
              (res.data.unilateralFlag == "1" && res.data.configLocationRejectFlag == "0")))
              {
                this.$router.push({ path: "/pei/amendment" });
              }else{
                this.$router.push({ path: "/pei/detail" });
              }

          }
        }
        catch(err) {
          Swal.mixin({
            toast: true,
            position: "top-end",
            showConfirmButton: false,
            timer: 6000
          }).fire(err.message, "Ha ocurrido un error mientras se procesaba la información.", "error");
        }
    },
    async updatePei() {
      let route = "/api/pei/updatePeiDetailsStep";
      if (this.PeiGeneralDB.unilateralFlag == "1") {
        route = "/api/pei/updatePsDetailsStep";
      }
      this.PeiGeneral.peiId = this.$store.getters.SelectedPei;
      return await new Promise(resolve => {
        this.$ApiPut(route, this.PeiGeneral)
          .then(res => {
            if (res && res.data != null) {
              resolve(res.data);

              //redirigir a detalle de PEI
              this.setPeiId(res.data.peiId);
              this.$router.push({ path: "/pei/detail" });
            }
          })
          .catch(function(error) {
            Swal.mixin({
              toast: true,
              position: "top-end",
              showConfirmButton: false,
              timer: 6000
            }).fire(error, "Ha ocurrido un error mientras se procesaba la información.", "error");
          });
      });
    },
    setPeiId(peiId) {
      this.$store.dispatch("setSelectedPei", peiId);
    },
    setLocationRejected(locationRejected) {
      this.$store.dispatch("setSelectedPeiLocationRejected", locationRejected);
    },
    convertDate(pickerDate) {
      let date = pickerDate.split("/");
      return new Date(date[2], months.indexOf(date[1]), date[0]);
    },
    toPeiDetail() {
			let peiId = this.$route.params.peiId;
			//llevar directamente a paso 3 de detall
			if (peiId) {
				for (let i = 0; i < 3; i++) {
					this.currentStep.active = false;
					this.currentStep.completed = true;
					this.currentStepIndex++;
					this.currentStep.active = true;
				}
				//set solo preguntas a mostrar
				let questionsToShow = [5.1, 5.2, 5.3, 6, 7.1];

        let step3 = this.getStepElement(4);
				step3.questions.forEach(question => {
					let questionElement = this.getQuestionElement(4, question.number);
					if (this.PeiGeneralDB.unilateralFlag == "1") {
						if (questionsToShow.includes(questionElement.number)) {
							questionElement.isVisible = true;
						} else {
							questionElement.isVisible = false;
						}
					} else {
						if (questionElement.number == 7.1) questionElement.isVisible = false;
					}
				});
			}
		},
    getCurrentYear() {
      return new Date().getFullYear();
    },
    toWizardLastStep() {
      this.getStep(5).confirmationTitle = "Se comenzará a crear la copia de los datos ingresados en el PEI o PS " + this.signedPeiId + "";

      //Llevar directamente a paso 4 de creación
      for (let i = 0; i < 3; i++) {
        this.currentStep.active = false;
        this.currentStep.completed = true;
        this.currentStepIndex++;
        this.currentStep.active = true;
      }
    },
    toWizardFirstStep() {
      this.getStep(5).confirmationTitle = "El PEI se comenzará a crear con la información detallada en las pantallas anteriores.";
      //Llevar directamente a paso 1 de creación
      for (let i = 4; i > 0; i--) {
        this.currentStep.active = false;
        this.currentStep.completed = false;
        this.currentStepIndex--;
        this.currentStep.active = true;
      }
    },
    parseAnswer(){

    },
    async uploadRejectedLocationDocument(peiId){

      let question5dot4Element = this.getQuestionElement(3, 9.4);

      if(question5dot4Element.selectedAnswer){
     
        let rejectDateField =  question5dot4Element.getField("fecha");
        let rejectedDate = moment(rejectDateField.value).locale("es").format("DD/MMM/YYYY");
        let documentField = question5dot4Element.getFieldElement("attachment");
        let document = {
              id: peiId,
              date: rejectedDate,
              file: documentField.value,
              attachmentId: documentField.attachmentId,
              status: documentField.status
            };
        await peiServices.uploadRejectedLocationDocument(document);
      }
    }
  }
};
</script>

<style scoped>
#widzard-header {
  min-height: 150px;
  background-color: white;
  width: 100%;
  border-bottom: 1px solid #cccccc;
}

#div-title {
  font-size: 4em;
}

#btn-cancell {
  top: 997px;
  left: 1143px;
  width: 124px;
  height: 38px;
  background: #c9545e 0% 0% no-repeat padding-box;
  opacity: 1;
  color: white;
}
#currentStep {
  font-size: 1.2em;
}

.wizard {
  height: calc(100vh - 166px);
}

.navbar-container {
  width: 100%;
  margin-top: 10px;
}

.step-pills {
  width: 100%;
}

.nav-tabs > li {
  width: 20%;
}
.step-pills > li {
  width: 20%;
}

#progress-bar {
  height: 3px;
  background: #2db782;
  width: 20%;
}

#section-tabs {
  background: #f2f2f2 0% 0% no-repeat padding-box;
  /* background-image: linear-gradient(rgba(206, 206, 206, 0.75), rgba(255, 255, 255, 0.74)); */
  padding: 15px 25px 25px;
  height: calc(100%);
}
.tab-pane-header {
  padding: 10px 70px;
  box-shadow: 0 10px 30px -12px rgba(224, 224, 224, 1), 0 4px 25px 0px rgba(224, 224, 224, 1), 0 8px 10px -5px rgba(224, 224, 224, 1);
}
.tag-badge {
  display: inline-flex;
  padding-right: 1em;
  height: 42px;
  background: transparent linear-gradient(90deg, #ffffff 0%, #e8e8e8 100%) 0% 0% no-repeat padding-box;
  border-radius: 26px;
  cursor: pointer;
}

.dot {
  width: 33px;
  height: 33px;
  line-height: 33px;
  border-radius: 50%;
  display: inline-block;
  margin-top: 2px;
  background: #ffffff 0% 0% no-repeat padding-box;
  box-shadow: 0px 3px 6px #00000029;
}
.orange-icon {
  color: #ff7058;
}

.tag-badge-title {
  text-align: left;
  font: Bold 25px/30px Lato;
  letter-spacing: 0;
  color: #11291b;
  display: inline-block;
}

.orange-icon-define {
  color: #ff7058;
  font-size: 25px;
}

.label-current {
  border-radius: 26px;
  background: transparent linear-gradient(90deg, #e2e2e2 0%, #ffffff00 100%) 0% 0% no-repeat padding-box;
  text-align: left;
  font: 900 20px Lato;
  color: #11291b;
  padding: 5px 14px;
  font-weight: 400;
	float: right; 
}

.tab-content {
  box-shadow: 0 10px 30px -12px rgba(0, 0, 0, 0.42), 0 4px 25px 0px rgba(0, 0, 0, 0.12), 0 8px 10px -5px rgba(0, 0, 0, 0.2);
  height: calc(100vh - 200px);
  overflow-x: auto;
}

.tab-pane-body {
  padding: 25px 40px;
  height: calc(100vh - 285px);
  overflow-y: auto;
}

.error-val {
  color: #ff5b5c;
  display: block;
}
.success-val {
  color: #2db782;
  display: block;
}
.warning-val {
  color: #af6708;
  display: block;
}

.footer-container {
  position: relative;
  bottom: 0;
  width: 100%;
  height: 60px; /* Set the fixed height of the footer here */
  line-height: 60px; /* Vertically center the text there */
  background-color: #f5f5f5;
}

.footer-menu {
  background: #ffffff 0% 0% no-repeat padding-box;
  height: 65px;
  box-shadow: 0 10px 30px -12px rgba(0, 0, 0, 0.42), 0 4px 25px 0px rgba(0, 0, 0, 0.12), 0 8px 10px -5px rgba(0, 0, 0, 0.2);
}

.footer-row {
  padding-top: 15px;
  padding-right: 25px;
  height: 45px;
}

.btn-footer {
  text-align: center;
  font: 500 16px/29px Lato;
  letter-spacing: 0;
  color: #ffffff;
  height: 38px;
  width: 130px;
  box-shadow: none;
  line-height: 20px;
}
.btn-footer.green {
  color: #2db782;
}

.green:hover, .green:hover:active {
  background-color: #2db782;
  color: #fff;
  border-color: #2db782;
}

.btn-footer:hover {
  color: white;
  opacity: 0.9;
}
.btn-back {
  background: #c9545e;
}
.btn-next {
  background: #2db782;
}

.custom-red-alert {
  background: #ffebeb 0% 0% no-repeat padding-box;
  border: 1px solid #ff5b5c;
  border-radius: 5px;
  font: 400 16px/24px Lato;
  letter-spacing: 0;
  color: #ff5b5c;
}

.custom-success-alert {
  color: #2db782;
  border: 1px solid #39da8a;
  border-radius: 4px;
}

.custom-warning-alert {
  background: #fff5e8 0% 0% no-repeat padding-box;
  border: 1px solid #fdac41;
  color: #af6708;
}
</style>
