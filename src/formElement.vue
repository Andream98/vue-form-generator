<template>
	<div class="form-element" 
		:class="[fieldRowClasses]"
		:style="field.wrapperStyle">
		<slot name="wrapper-hook"
			:field="field"
			:getValueFromOption="getValueFromOption"
		/>
		<label
			v-if="fieldTypeHasLabel"
			:for="fieldID"
			:style="field.labelStyle"
			:class="field.labelClasses">
			<slot name="label" 
				:field="field"
				:fieldID="fieldID"
				:fieldComponent="fieldComponent"
				:getValueFromOption="getValueFromOption"
			/>
			<slot name="help" 
				:field="field" 
				:getValueFromOption="getValueFromOption"
			/>
		</label>
		<div class="field-wrap" :style="field.fieldStyle">
			<component
				ref="child"
				v-if="fieldFound"
				:is="fieldType"
				:model="model"
				:schema="field"
				:form-options="options"
				:event-bus="eventBus"
				:fieldID="fieldID"
				@field-touched="onFieldTouched"
				@errors-updated="onChildValidated"
			/>
			<div v-else class="field-not-found">
				<slot name="field-not-found"
					:field="field"
					:getValueFromOption="getValueFromOption"
				>
					Field not found
				</slot>
			</div>
			<div
				v-if="buttonsAreVisible"
				class="buttons">
				<button
					v-for="(btn, index) in field.buttons"
					@click="buttonClickHandler(btn, field, $event)"
					:class="btn.classes"
					:key="index"
					v-text="btn.label"
				/>
			</div>
		</div>

		<template v-if="fieldHasHint">
			<slot name="hint" 
				:field="field" 
				:getValueFromOption="getValueFromOption"
			/>
		</template>

		<template v-if="fieldHasErrors">
			<slot
				name="errors"
				:childErrors="childErrors"
				:field="field"
				:getValueFromOption="getValueFromOption"
			/>
		</template>
	</div>
</template>
<script>
import isNil from "./utils/isNil";
import { slugifyFormID } from "./utils/schema";
import formMixin from "./formMixin.js";
import { resolveComponent } from 'vue';

export default {
	name: "form-element",
	mixins: [formMixin],
	props: {
		model: {
			type: Object,
			default() {
				return {};
			}
		},
		options: {
			type: Object,
			default() {
				return {};
			}
		},
		field: {
			type: Object,
			required: true
		},
		errors: {
			type: Array,
			default() {
				return [];
			}
		},
		eventBus: {
			type: Object,
			required: true,
		}
	},
	data() {
		return {
			childErrors: [],
			childTouched: false
		};
	},
	computed: {
		fieldComponent() {
			return this;
		},
		fieldID() {
			const idPrefix = this.options.fieldIdPrefix || "";
			return slugifyFormID(this.field, idPrefix);
		},
		// Get type of field 'field-xxx'. It'll be the name of HTML element
		fieldType() {
			return "field-" + this.field.type;
		},
		fieldFound() {
			if(this.field && this.field.type) {
				const capitalized = this.field.type.charAt(0).toUpperCase() + this.field.type.slice(1)
				return !!resolveComponent('Field' + capitalized);
			}
		},
		// Should field type have a label?
		fieldTypeHasLabel() {
			if (isNil(this.field.label)) {
				return false;
			}
			let fieldOptions = this.getValueFromOption(this.field, "fieldOptions");
			let condition = this.field.type === "input" && !isNil(fieldOptions);
			let relevantType = condition ? fieldOptions.inputType : this.field.type;
			const typeWithoutLabel = ["button", "submit", "reset"];

			return !typeWithoutLabel.includes(relevantType);
		},
		fieldHasHint() {
			return !isNil(this.field.hint);
		},
		fieldHasErrors() {
			return this.childErrors.length > 0;
		},
		fieldRowClasses() {
			let baseClasses = {
				[this.options.validationErrorClass || "error"]: this.fieldHasErrors,
				[this.options.validationSuccessClass || "valid"]: !this.fieldHasErrors && this.childTouched,
				[this.options.validationCleanClass || "clean"]: !this.fieldHasErrors && !this.childTouched,
				disabled: this.getValueFromOption(this.field, "disabled"),
				readonly: this.getValueFromOption(this.field, "readonly"),
				featured: this.getValueFromOption(this.field, "featured"),
				required: this.getValueFromOption(this.field, "required")
			};

			baseClasses = this.getStyleClasses(this.field, baseClasses);

			if (!isNil(this.field.type)) {
				baseClasses["field-" + this.field.type] = true;
			}

			return baseClasses;
		},
		buttonsAreVisible() {
			return Array.isArray(this.field.buttons) && this.field.buttons.length > 0;
		}
	},
	methods: {
		getValueFromOption(field, option, defaultValue = false) {
			if (typeof field[option] === "function") {
				return field[option].call(this, this.model, field, this);
			}

			if (isNil(field[option])) {
				return defaultValue;
			}
			return field[option];
		},

		buttonClickHandler(btn, field, event) {
			return btn.onclick.call(this, this.model, field, event, this);
		},
		onFieldTouched() {
			this.childTouched = true;
		},
		onChildValidated(errors) {
			this.childErrors = errors;
		}
	}
};
</script>
<style>
.form-element:not([class*=" col-"]) {
	width: 100%;
}
.form-element {
	display: inline-block;
	vertical-align: top;
	margin-bottom: 1rem;
}
.form-element label {
	font-weight: 400;
}
.form-element label > :first-child {
	display: inline-block;
}

.form-element.featured > label {
	font-weight: bold;
}

.form-element.required > label:after {
	content: "*";
	font-weight: normal;
	color: Red;
	padding-left: 0.2em;
	font-size: 1em;
}

.form-element.disabled > label {
	color: #666;
	font-style: italic;
}

.form-element.error input:not([type="checkbox"]),
.form-element.error textarea,
.form-element.error select {
	border: 1px solid #f00;
	background-color: rgba(#f00, 0.15);
}

.form-element.error .errors span {
	display: block;
	background-image: url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAiklEQVR4Xt2TMQoCQQxF3xdhu72MpZU3GU/meBFLOztPYrVWsQmEWSaMsIXgK8P8RyYkMjO2sAN+K9gTIAmDAlzoUzE7p4IFytvDCQWJKSStYB2efcAvqZFM0BcstMx5naSDYFzfLhh/4SmRM+6Agw/xIX0tKEDFufeDNRUc4XqLRz3qabVIf3BMHwl6Ktexn3nmAAAAAElFTkSuQmCC");
	background-repeat: no-repeat;
	padding-left: 17px;
	padding-top: 0px;
	margin-top: 0.2em;
	font-weight: 600;
}
.form-element.error .errors {
	color: #f00;
	font-size: 0.8em;
}
</style>
