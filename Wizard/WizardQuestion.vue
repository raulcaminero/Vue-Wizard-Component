<template>
  <div class="d-flex flex-column">
    <div class="question-element"
         v-if="isVisible">
      <div class="item-pane" :class="{'no-border': noBorder}">
        <div class="question-box"
             v-if="hideIconBox">
          <div class="box-number">
            <div v-if="hasIcon">
              <i :class="icon"/>
            </div>
            <div v-else>
              {{ subNumber == undefined ? displayNumberText : displayNumber }}<sup class="question-number-sup">{{ subNumber }}</sup>
            </div>
          </div>
        </div>
        <div class="question-line col">
          <p class="mb-0 p-2 pl-4"
             v-html="text"
             :class="{'border-bottom':hasFields }"></p>
          <Error v-if="(hasAWarning || hasAFieldError)" 
                 :is-alert-mode="hasAWarning"
                 :message="validationMessage"/> 
          <!-- TextArea -->
          <div v-if="isTextArea">
            <textarea class="form-control textarea"
                      name=""
                      cols="30"
                      rows="5"
                      v-model="questionOpenAnswer"
                      maxlength="3000"
                      :disabled="disabled"></textarea>
            3,000 caracteres
          </div>

          <slot name="children" />

          <div v-if="showCommentsBox">
            <label v-if="isYesNoAnswer">Explique:</label>
            <textarea class="form-control textarea"
                      name=""
                      cols="30"
                      rows="5"
                      v-model="questionOpenAnswer"
                      maxlength="3000"
                      :disabled="disabled"></textarea>
          </div>

          <!-- END TextArea -->

          <div class="col pl-0"
               v-if="hasFields">
            <div class="mt-2 mb-2 p-2 pl-4"
                 v-if="isVisibleFieldCaption"
                 v-html="fieldCaption"></div>
            <div class="row p-2 pl-4">
              <div class="col-12">
                <div class="row">
                  <div v-for="(field,index) in questionFields"
                       :key="index"
                       :class="field.class"
                       v-show="field.visible || (field.visible == null && (field.showWithValue == null || field.showWithValue == selectedAnswer || (field.listValuesForShow != null && field.listValuesForShow.some(item => item === selectedAnswer))))">
                    <div >
                      <div class="field-label"
                           v-show="field.label != null">
                        <p v-html="field.label"></p>
                      </div>

                      <custom-drop-down 
                        :id="field.id"
                        :options="field.options"
                        @input="onFieldChange($event,field)"
                        :fieldkey="field.fieldkey"
                        :fieldname="field.fieldname"
                        :placeholder="field.placeholder"
                        :error="field.error"
                        :disabled="field.disabled || disabled"
                        :custom-error-msg="field.customErrorMsg"
                        v-if="field.type=='dropdown'"
                        v-model="field.value"
                        :ref="`question_field_${field.id}`"/>

                      <custom-date-picker
                        :placeholder="field.placeholder"
                        v-model="field.value" 
                        v-if="field.type=='date'"
                        :max-date="field.maxDate"
                        :ref="`question_field_${field.id}`"
                        :disabled="field.disabled || disabled"/>	

                      <custom-file-input 
                        :id="field.id"
                        :name="field.name"
                        v-model="field.value" 
                        v-if="field.type=='file'" 												
                        :ref="`question_field_${field.id}`"
                        :disabled="field.disabled || disabled"
                        :attachment-id="field.attachmentId"
                        :attachment-status="field.attachmentStatus"/>

                      <div class="comment-text-area" 
                           v-if="field.type=='textarea'">
                        <custom-text-area
                          :id="field.id"
                          :maxlength="field.maxChars"
                          :placeholder="field.placeholder"
                          :classes="['eq-field']"
                          :count-classes="['text-secondary']"
                          v-show="true"
                          v-model="field.value"
                          :ref="`question_field_${field.id}`"
                          :disabled="field.disabled || disabled"/>
                      </div>
                      <div class="mt-2"
                           v-if="field.type=='multiple-selection-options'">
                        <div 
                          v-for="(option, index) in field.options" 
                          :key="`chk${field.id}${option.Value}_${index}`" 
                          class="custom-control custom-checkbox mr-sm-2">
                          <input 
                            :id="`chkWizardQuestion${number}${field.id}${index}`" 
                            type="checkbox" 
                            class="custom-control-input" 
                            :value="option.Value" 
                            v-model="field.selectedOptions" 
                            @change="field.onChange($event, field)" 
                            :ref="`question_field_${field.id}`"
                            :disabled="field.disabled || disabled">
                          <label 
                            class="custom-control-label font" 
                            :for="`chkWizardQuestion${number}${field.id}${index}`" >
                            {{ option.Display }}
                          </label>
                        </div>
                      </div>
                    </div>						
                  </div>
                </div>
              </div>
            </div>
		
            <span v-if="(hasAFieldError || hasAFieldWarning)"
                  :class="{'error-val': hasAFieldError, 'warning-val': hasAFieldWarning}">
              <i class="fa fa-times-circle"
                 aria-hidden="true"></i>
              <span v-html="validationMessage"
                    class="ml-1"></span>

            </span>
          </div>
        </div>



        <!-- RadioButtons -->
        <div class="align-items-end d-flex p-0 custom-radio-container"
             v-if="isYesNoAnswer || isConditionalMultipleSelection || isYesNoNaAnswers">
          <div class="w-49 ml-2"
               v-for="index in columnOffset"
               :key="'bcolumnOffset'+index"
               v-show="hasOffsetColumn"></div>
          <div
            v-for="(item, index) in questionOptions"
            :key="index"
            :id="`rb_${number}`"
            class="text-center rounded-top bg-gray mb-0 custom-radio-button"
            :class="{ 'bg-green': (selectedAnswer == item.Value) }">
            <div class="radio">
              <label :id="`lbl_${number}`"
                     class="w-100 mb-0" >
                <input
                  type="radio"
                  :name="`Question${number}YesNoOption`"
                  @change="onValueChange"
                  :value="item.Value"
                  v-model="selectedAnswer"
                  :disabled="disabled">
                <span class="cr rounded-circle"><i class="cr-icon fa fa-check text-white"
                                                   :class="{'show-check':(selectedAnswer == item.Value)}"></i></span>
              </label>
            </div>
            <span class="step-label">{{ item.Display }}</span>
          </div>
          <div class="w-49"
               v-for="index in columnOffset"
               :key="'fcolumnOffset'+index"
               v-show="!hasOffsetColumn"></div>
        </div>
        <!-- END RadioButtons -->

        <!-- Dropdown -->
        <div v-if="isDropDown"
             class="col-md-4">
          <div class="mt-2">
            {{ dropdownUpperText }}
          </div>
          <select class="form-control question-select mt-1"
                  @change="onValueChange($event)"
                  v-model="selectedAnswer"
                  :disabled="disabled">
            <option v-for="(option, index) in questionOptions"
                    :key="'isDropDown'+index"
                    :value="option.Value">
              {{ option.Display }}
            </option>
          </select>
        </div>
        <!-- END Dropdown -->

        <!-- INPUT -->
        <div v-if="isOpenTextBox"
             class="col-md-2">
          <input class="form-control question-answer mt-3"
                 type="text"
                 v-model="questionOpenAnswer"
                 :disabled="disabled">
        </div>
        <!-- END INPUT -->

        <div v-if="isDate"
             class="mt-3">
          <custom-date-picker
            right
            :format="'dd/MMM./yyyy'"
            @onSelectedDate="onValueChange"
            v-model="selectedAnswer" 
            :disabled="disabled"/>
        </div>
      </div>
      <!-- PANEL DE INFORMACION -->
      <div class="item-pane-info"
           v-if="notes && notes.length > 0">
        <div class="row justify-content-md-center">
          <div class="col-10">
            <p v-for="(note, index) in notes"
               :key="`info_item_${index}`">
              <i class="far fa-exclamation-circle"></i>
              <span v-html="note.message"></span>
            </p>
          </div>
        </div>
      </div>
      <!-- FIN PANEL DE INFORMACION -->
    </div>
  </div>
</template>

<script>
// import { forestgreen } from 'color-name';
// import { log } from 'util';
//ECG:Para tipos de seleccion excepcionales se debera crear un tipo de pregunta nuevo? ej. Region-Distrito-Escuela
const answerControlTypes = Object.freeze({ Dropdown: 0, YesNo: 1, TextBox: 2, TextArea: 3, Date: 4, ConditionalMultipleSelection: 5, MultipleSelection: 6, QuestionForm: 7,YesNoNa:8 });
// const validationResultTypes = Object.freeze({ Error: 1, Warning: 2 });

import CustomDatePicker from '../../components/common/CustomDatePicker';
import CustomFileInput from '../../components/common/CustomFileInput';
import CustomTextArea from '../../components/common/CustomTextArea';
import CustomDropDown from '../../components/common/CustomDropDown';
import Error from '../../components/common/Error';

const defaultYesNoAnswers = [
	{ Display: "Si", Value: true },
	{ Display: "No", Value: false }
];
export default {
	name: "WizardQuestion",	
	components:{
		CustomDatePicker,
		CustomFileInput,
		CustomTextArea,
		CustomDropDown,
		Error
	},
	props: {
		defaultOptionValueSelected: {
			type: Number,
			default: 0
		},
		number: {
			type: Number,
			default: 0
		},
		fields: {
			type:Array,
			default: () => new Array()
		},
		displayNumber: {
			type:String,
			default:""
		},
		subNumber: {
			type:String,
			default:""
		},
		description: {
			type:String,
			default:""
		},
		warningMessage: {
			type:String,
			default:""
		},
		errorMessage: {
			type:String,
			default:""
		},
		answerControlType: {
			type: Number,
			default: 0
		},
		optionListValue: {
			type:Array,
			default: () => new Array()
		},
		dropdownUpperText: {
			type:String,
			default:""
		},
		getOptions: Function,
		getDefaultValue: Function,
		onChange: Function,
		isRequired: Boolean,
		requireErrorCode: {
			type:String,
			default:""
		},
		requireErrorMessage: {
			type:String,
			default:""
		},
		validator: Function,
		pickerMinDate:{
			type:String,
			default:""
		},
		fieldCaption:{
			type:String,
			default:""
		},
		visible: {
			type: Boolean,
			default: true
		},
		selectedOption: {
			type:String,
			default:""
		},
		showMultipleSelectionOptions: {
			type: Boolean,
			default: false
		},
		showCommentsBox: {
			type: Boolean,
			default: false
		},
		initialState: Boolean,
		notes: {
			type:Array,
			default: () => new Array()
		},
		showWithValue: {
			type:String,
			default:""
		},
		isDisabled: {
			type: Boolean,
			default: false
		},
		hasIcon: {
			type: Boolean,
			default:()=> false
		}, 
		icon: {
			type:String,
			default:""
		},
		hideIconBox:{
			type: Boolean,
			default: ()=> true
		},
		columnOffset:{
			type: Number,
			default: 0
		},
		showFieldCaption: {
			type: Boolean,
			default: true
		},
		noBorder: {
			type: Boolean,
			default: false
		}
	},
	data() {
		return {	
			
			isChecked: false,
			isCheckedYesNoOption: false,
			disabled: false,
			selectedAnswer: undefined,
			selectedOptions: [],
			questionOpenAnswer: undefined,
			validationResult: {
				isValid: true,
				resultType: 0,
				messageText: "",
				showFieldValidation: false
			},
			invalidSelection: false,
			validationResultTypes: { Ok: 0, Error: 1, Warning: 2, FancyWarning: 3 },
			options: null,
			multipleSelectionOptions: null,
			validators: [],
			isVisible: true,
			text: "",
			infos: [],
			questionFields:this.fields,
			controlType: null,
			isVisibleFieldCaption: this.showFieldCaption,			
		};
	},
	mounted() {
		this.controlType = this.answerControlType;

		if(this.isDisabled != null){
			this.disabled = this.isDisabled
		}

		if(this.defaultOptionValueSelected != undefined && this.defaultOptionValueSelected != 0 ){
			this.selectedAnswer = this.defaultOptionValueSelected;
		}

		if (this.visible != null) {
			this.isVisible = this.visible;
		}
		this.text = this.description;
		if (this.options == null) {
			this.refreshOptions();

			//cargando valor por defecto.
			if (this.getDefaultValue != null) {
				this.getDefaultValue().then(defaultValue => {
					this.selectedAnswer = defaultValue;
					this.$set(this, "selectedAnswer", defaultValue);
					this.onValueChange();
				});
			}
		}
		
		if(this.options == null && this.optionListValue != null){
			this.options = this.optionListValue;
		}

		if (this.isRequired == true) {
			this.validators.push(this.validateRequired);
		}

		if (this.validator != null) {
			this.validators.push(this.validator);
		}
	},
	watch: {
		isVisible() {
			this.clear();
		},
		invalidSelection: {
			handler: function() {
				this.$emit("questionChange", this);
			},
			deep: true
		},
		selectedAnswer:function(newValue, oldValue){
			if(this.fields!=null && oldValue != null ){
				this.fields.forEach(field => {
					if(field.showWithValue == newValue || (field.listValuesForShow != null && field.listValuesForShow.some(item => item === newValue))){
						field.value = null;				
						if(field.type == "multiple-selection-options"){
							field.selectedOptions = [];
						}
					}
				});
			}
		}
	},
	computed: {
		optionsCountClass(){
			if(this.questionOptions !=null)
				return "col-md-"+(this.questionOptions.length+this.columnOffset); 
			return null;
		},
		hasOffsetColumn(){
			return this.questionOptions && this.questionOptions.length>2;
		},
		hasFields(){
			if(this.questionFields && this.questionFields.length>0){
				return  this.questionFields.some(field=>{
					let isFieldWithSameSelectedValue = field.showWithValue != null && field.showWithValue == this.selectedAnswer;
					return isFieldWithSameSelectedValue || field.showWithValue == null;
				});
			}
			
			return false;
		},
		hasAnError() {
			return !this.hasAnWarning && ((this.invalidSelection || (!this.validationResult.isValid && this.validationResult.resultType == this.validationResultTypes.Error)));
		},
		hasAWarning() {
			return this.validationResult.resultType == this.validationResultTypes.Warning;
		},
		hasAFieldWarning() {
			return this.validationResult.showFieldValidation  && this.validationResult.resultType == this.validationResultTypes.Warning;
		},
		hasAFieldError(){
			return this.validationResult.showFieldValidation && 
			!this.validationResult.isValid && 
			this.validationResult.resultType == this.validationResultTypes.Error;
		
		},
		validationMessage() {
			if (this.validationResult.messageText == null) {
				if (this.isYesNoAnswer) return "Pregunta "+this.number+": Favor de seleccionar Sí o No.";
			}
			return this.validationResult.messageText;
		},
		displayNumberText: function() {
			if (this.displayNumber == null) {
				let displayNumber = Math.floor(this.number);
				if (this.subNumber) return displayNumber + this.subNumber;
				else return displayNumber;
			}
			if (this.subNumber) return this.displayNumber + this.subNumber;
			return this.displayNumber;
		},
		selectedAnswerId: function() {
			let result = this.options.filter(element => element.Value == this.selectedAnswer);
			if (result.length > 0) {
				return result[0].Id;
			}
			return null;
		},
		selectedAnswerText: function() {
			let result = this.options.filter(element => element.Value == this.selectedAnswer);
			if (result.length > 0) {
				return result[0].Display;
			}
			return null;
		},
		questionOptions: function() {
			if (this.isYesNoAnswer || this.isConditionalMultipleSelection) {
				return defaultYesNoAnswers;
			}
			return this.options;
		},
		isYesNoAnswer: function() {
			return answerControlTypes.YesNo == this.answerControlType;
		},
		isDropDown: function() {
			return answerControlTypes.Dropdown == this.answerControlType;
		},
		isOpenTextBox: function() {
			return answerControlTypes.TextBox == this.answerControlType;
		},
		isTextArea: function() {
			return answerControlTypes.TextArea == this.answerControlType;
		},
		isDate: function() {
			return answerControlTypes.Date == this.answerControlType;
		},
		isConditionalMultipleSelection: function() {
			return answerControlTypes.ConditionalMultipleSelection == this.answerControlType;
		},
		isMultipleSelection: function() {
			return answerControlTypes.MultipleSelection == this.answerControlType;
		},
		isYesNoNaAnswers: function() {
			return answerControlTypes.YesNoNa == this.answerControlType;
		},
		isQuestionForm: function(){
			return this.answerControlType == answerControlTypes.QuestionForm
		}
	},
	methods: {
		onValueChange() {
			if (this.onChange != null) {
				this.onChange(this);
			}
			if (!this.isDate)
				//fecha es validada en onChange de la pregunta
				this.validate(true);
		},
		validateRequired: function(question, isOnchangeEvent) {

			if(question != null && !isOnchangeEvent){
				
				let hasFields = this.fields != null && this.fields.length > 0;
				let isQuestionForm = (this.selectedAnswer != null && hasFields) || this.isQuestionForm;

				if(hasFields && isQuestionForm){									
					let invalidField = this.questionFields.find(field => {
								
						let isVisible = field.visible || (field.visible == null && (field.showWithValue == null  || field.showWithValue == this.selectedAnswer || (field.listValuesForShow != null && field.listValuesForShow.some(item => item === this.selectedAnswer))  ));
						let isFieldRequired = field.required && isVisible;

						let isMultipleSelectionField = field.type == "multiple-selection-options";
						if(isFieldRequired){
							if(isMultipleSelectionField){
								let isInvalidSelection = (field.selectedOptions == null || (field.selectedOptions != null && field.selectedOptions.length ==0));										
								return isInvalidSelection;
							}
							if(field.type =="file" ){
								let fieldElement = this.getFieldElement(field.id);
								let isDeletedAttachment = (fieldElement.attachmentId!=null && fieldElement.status == "deleted");
								let isNotLoadedAttachment = ((fieldElement.value == null || fieldElement.value == "") && 
											 (fieldElement.attachmentId == null));
											 
								return (isNotLoadedAttachment || isDeletedAttachment);
							}
													
							return field.value == null || field.value == "";							 
						}
						return false;																				
					});

					if(invalidField != null)
					{						
						this.validationResult.isValid = false;
						this.validationResult.resultType = this.validationResultTypes.Error;							
						if (invalidField.requireErrorMessage != null) {
							this.validationResult.messageText = invalidField.requireErrorMessage;
							this.validationResult.showFieldValidation = invalidField.showFieldValidation;
						}
						return false;
					}					
				}
			}

			if (this.isRequired) {			
				if (
					((this.isDropDown || this.isDate || this.isYesNoAnswer || this.isYesNoNaAnswers) && this.selectedAnswer == null) || //pregutnas si o no, dropdown, fecha
					(this.showCommentsBox && (this.questionOpenAnswer == null || this.questionOpenAnswer == "")) || //comentario en diferentes casos
					(this.isOpenTextBox && (this.questionOpenAnswer == "" || this.questionOpenAnswer == null)) || //input
					((this.showMultipleSelectionOptions || this.isMultipleSelection) && this.selectedOptions.length == 0) ||
					(this.isTextArea && (this.questionOpenAnswer == null || this.questionOpenAnswer == "")) ||
					(this.isConditionalMultipleSelection && (this.selectedAnswer == null && !isOnchangeEvent))
				) {
					
					//selección múltipe
					this.validationResult.isValid = false;
					this.validationResult.resultType = this.validationResultTypes.Error;
					if (this.requireErrorMessage != null) {
						this.validationResult.messageText = this.requireErrorMessage;
					}
					return false;
				}	
			}
			return true;
		},
		validate(isOnChangeEvent) {
			if (this.isVisible) {
				// si la opción dispara warning
				if (this.invalidSelection && this.validationResult.resultType == 2) {
					return true;
				}

				if (this.invalidSelection) {
					return false;
				}

				//Si la opcion seleccionada no dispara error, entonces ejecuta las demas validaciones.
				if (this.validators.length > 0) {
					let result = true;
					this.validators.forEach(validator => {
						if (!validator(this, isOnChangeEvent)) {
							result = false;
						}
					});
					this.validationResult.isValid = result;
					if (result) {
						this.validationResult.resultType = this.validationResultTypes.Ok;
						this.validationResult.messageText = "";
					}
				} else {
					this.validateRequired(this, isOnChangeEvent);
				}
				return this.validationResult.isValid;
			}
			return true;
		},
		clear() {
			this.selectedAnswer = undefined;
			this.questionOpenAnswer = "";
			this.selectedOptions = [];
			this.invalidSelection = false;

			let fieldComment = this.getField("comentario");  
			if(fieldComment){        
				fieldComment.visible = false;
			}
			this.validationResult.isValid = true;
			this.validationResult.resultType = this.validationResultTypes.Ok;

			if(this.fields != null){
				this.fields.forEach(field => {
					field.value = null;					
					if(field.type == "multiple-selection-options"){
						field.selectedOptions = [];
					}
				});
			}
		},
		ThrowErrorMessage() {
			this.$parent.$parent.showMessage(this.requireErrorMessage, "alert-danger");
		},
		CambioStatus() {
			this.$parent.$parent.CambioStatus();
		},
		setVisible(status) {
			this.isVisible = status;
		},
		refreshOptions() {
			//Cargando opciones de pregunta
			if (this.getOptions != null) {
				this.getOptions().then(options => {
					this.options = options;
					this.multipleSelectionOptions = options;
				});
			}

			//Actualizando campos
			if(this.fields != null){
				this.questionFields.forEach(field => {
					if(field.getOptions !=null ){
						field.getOptions().then(options => {
								field.options = options;
								field.value = null;
						});					
					}
				});
			}
		},
		getFieldElement(id){
			if(this.$refs["question_field_"+id]!=null)
				return this.$refs["question_field_"+id][0];
			return null;
		},
		getField(id){
			if(this.questionFields){				
				return this.questionFields.find(field => field.id == id);
			}
		},
		onFieldChange(value, field){
			if(field != null && field.onChange !=null ){
				field.onChange(value,this);
			}
		}
	}
};
</script>

<style scoped>
.show-check {
	opacity: 1 !important;
}

.item {		
	display: flex;		
	justify-content: space-between;
}

.question{
	box-shadow: -3px 1px 3px #00000029;
	align-content: space-between;
}

.item-pane {
	justify-content: space-between;
	min-width: 50%;
	box-shadow: -3px 1px 3px #00000029;
	margin-top: 10px;
	display: flex;
	border-bottom: solid 1px #e2e2e2;
}

.item-pane.no-border {
	border-bottom: none;
}

.item-pane-info {
    background: #F2F2F2 0% 0% no-repeat padding-box;
    border-radius: 0px 0px 5px 5px;
    margin-bottom: 10px;
    font: 400 14px/17px Lato;
    letter-spacing: 0;
    color: #11291B;
    padding: 20px 10px 10px;
}

.item-pane-info p {
    margin-bottom: 10px;
}

.item-pane-info p i {
    margin-right: 8px;
    font-size: 16px;
}

.question-box {
	width: 65px;
	background: #ff7058 0% 0% no-repeat padding-box;
	border-radius: 5px 5px 0px 0px;
	text-align: center;
	color: #ffffff;
	letter-spacing: 0;
	display: flex;
	align-items: center;
	justify-content: center;
	/* padding: 22px 12px; */
}

.box-number {
	font: Bold 32px/19px Lato;
}
.box-label {
	font: 15px Lato;
	display: block;
}
.question-line.pl-55 {
	padding-left: 55px !important;
}
.question-line p {
	padding-bottom: 0px;
}

.question-line p >>> p {
	margin-bottom: 0px !important;
}

.w-49 {
	width: 49%;
}
.w-100 {
	width: 100% !important;
}
.bg-gray {
	background: #f2f2f2;
}
.bg-green {
	background: #2db782;
}
/************* end of custom radio *****************/
.checkbox label:after,
.radio label:after {
	content: "";
	display: table;
	clear: both;
}

.checkbox .cr,
.radio .cr {
	position: relative;
	display: inline-block;
	border: 2px solid #a9a9a9;
	border-radius: 0.25em;
	width: 1.2em;
	height: 1.2em;
	margin-top: 4px;
}

.cr {
	border-radius: 50%;
}

.checkbox .cr .cr-icon,
.radio .cr .cr-icon {
	position: absolute;
	font-size: 0.2em;
	line-height: 0;
	top: 53%;
	left: 13%;
}

.cr-icon {
	margin-top: 0.04em;
}

.checkbox label input[type="checkbox"],
.radio label input[type="radio"] {
	display: none;
}

.checkbox label input[type="checkbox"] + .cr > .cr-icon,
.radio label input[type="radio"] + .cr > .cr-icon {
	opacity: 0;
}

.checkbox label input[type="checkbox"]:checked + .cr > .cr-icon,
.radio label input[type="radio"]:checked + .cr > .cr-icon {
	opacity: 1;
}

.checkbox label input[type="checkbox"]:disabled + .cr,
.radio label input[type="radio"]:disabled + .cr {
	opacity: 0.5;
}
.rounded-top {
	height: 100%;
	display: flex;
	flex-direction: column;
	justify-content: center;
}
/************* end of custom radio *****************/
label,
p {
	font-weight: normal;
}

.step-label {
	display: block;
	font: Regular 11px/11px Lato;
	font-size: 10px;
	letter-spacing: 0;
	color: #c9c9c9;
}

.bg-green .step-label {
	color: white;
}

.bg-green .radio .w-100 .cr.rounded-circle {
	border: 2px solid white;
	opacity:1;
}

.step-label.active {
	color: #2db782;
	font-weight: bold;
}
.error-val {	
	color: #ff5b5c;
	display: block;
	padding-left: 25px;
}
.success-val {
	color: #2db782;
	display: block;
}
.warning-val {
	color: #af6708;
	display: block;
}
.question-number-sup {
	font: 21px Lato;
	letter-spacing: 0;
	color: #ffffff;
}
.form-control.textarea {
	background: #FFFFFF 0% 0% no-repeat padding-box;
	border: 1px solid #E2E2E2;
	border-radius: 5px;
}
.custom-control-label.font {
	font: 400 16px/19px Lato;
	color: #11291B;
	padding-top: 4px;
}
.custom-control-input:checked~.custom-control-label::before {
	color: #fff;
	border-color: #2DB782;
	background-color: #2DB782;
}
.custom-red-alert {
	background: #FFEBEB 0% 0% no-repeat padding-box;
	border: 1px solid #FF5B5C;
	border-radius: 5px;
}
.question-comment{
	background-color: #F2F2F2 ;
	font-weight: normal;
	text-align: left;
	font: Regular 14px/17px Lato;
	letter-spacing: 0;
	color: #707070;
	opacity: 1;
	padding-left: 2%;
	padding-right: 2%;
	margin-top: 0px;
}

 .comment-text-area{
    padding:20px;
	background: transparent linear-gradient(180deg, #E4E6E2 0%, #F4F6F300 100%) 0% 0% no-repeat padding-box;
	
  }

	.custom-radio-container {
		padding-top: 15px !important;
		margin-right: 42px;
	}

	.custom-radio-container .custom-radio-button {
		width: 70px;
		margin-right: 8px;
	}

</style>
