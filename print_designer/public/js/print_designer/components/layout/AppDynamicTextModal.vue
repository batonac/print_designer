<template>
	<AppModal
		v-bind="{ size }"
		:backdrop="true"
		:isDraggable="false"
		@cancelClick="cancelClick"
		@primaryClick="primaryClick"
	>
		<template #title>
			<span v-if="table">Table ({{ table.label || table.fieldname }})</span>
			<span v-else>
				{{ selectedDoctypeLabel }} &nbsp;
				<small> (Dynamic Text)</small>
			</span>
		</template>
		<template #body>
			<div class="dynamic-results">
				<div class="side-section" v-if="!table">
					<input
						class="searchbar"
						:placeholder="__('Search Doctypes')"
						v-model="search_link"
						@input="
							frappe.utils.debounce((e) => {
								search_link = e.target.value;
							}, 300)
						"
					/>
					<ul class="sidebar">
						<li
							v-for="field in MainStore.getLinkMetaFields(
								search_link,
								previewRef?.parentField
							)"
							@click="handleSidebarClick(field.fieldname)"
							:value="field.fieldname"
							class="sidebar-item"
							:class="{
								'item-selected': previewRef?.parentField == field.fieldname,
							}"
						>
							<span>{{ field.label }}</span>
						</li>
					</ul>
				</div>
				<div class="main-section">
					<div class="searchbar-sticky">
						<input
							v-focus
							class="searchbar"
							:placeholder="__('Search Field by label, fieldname or fieldtype')"
							v-model="search_text"
							@input="
								frappe.utils.debounce((e) => {
									search_text = e.target.value;
								}, 300)
							"
						/>
					</div>
					<div class="container-main">
						<div
							v-for="(fields, fieldtype) in MainStore.getTypeWiseMetaFields({
								selectedParentField: previewRef?.parentField,
								selectedTable: table,
								search_string: search_text,
							})"
							:key="fieldtype"
						>
							<div class="fieldtype-title">
								{{ fieldtype }}
							</div>
							<div class="fields">
								<div
									class="field"
									v-for="field in fields"
									:key="field.fieldname"
									:value="field.fieldname"
								>
									<div
										class="field-container"
										@click="selectField(field, fieldtype)"
										:class="{
											'field-selected': fieldnames.filter(
												(el) =>
													el.parentField == previewRef?.parentField &&
													el.fieldname == field.fieldname
											).length,
										}"
									>
										<div class="field-label">
											<span>{{ field.label || field.fieldname }}</span>
											<span class="fa fa-check-circle icon-show"></span>
										</div>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>
			</div>
			<AppDynamicPreviewModal
				v-bind="{ fieldnames, modalLabel: openDynamicModal.label, isTable: !!table }"
				:ref="
					(childRef) => {
						childRef && (previewRef = childRef);
					}
				"
			/>
		</template>
	</AppModal>
</template>
<script setup>
import { ref, onMounted, watch, nextTick } from "vue";
import { getMeta, getValue } from "../../store/fetchMetaAndData";
import { useMainStore } from "../../store/MainStore";
import AppModal from "./AppModal.vue";
import AppDynamicPreviewModal from "./AppDynamicPreviewModal.vue";
const MainStore = useMainStore();
const props = defineProps({
	openDynamicModal: {
		type: Object,
		required: true,
	},
	table: {
		type: Object,
		default: null,
	},
});
const search_link = ref("");
const search_text = ref("");
const doctype = ref("");
const selectedDoctypeLabel = ref("");
const fieldnames = ref([]);
const previewRef = ref(null);

const parentFieldWatcher = watch(
	() => previewRef.value?.parentField,
	() => {
		if (previewRef.value == null) {
			parentFieldWatcher();
			return;
		}
		if (!previewRef.value.parentField) {
			doctype.value = "";
			selectedDoctypeLabel.value = MainStore.doctype;
		} else {
			let meta = MainStore.metaFields.find(
				(field) => field.fieldname == previewRef.value.parentField
			);
			selectedDoctypeLabel.value = meta.label || meta.fieldname;
			if (!meta.childfields) {
				getMeta(meta.options, previewRef.value.parentField);
			}
			doctype.value = meta.options;
		}
	}
);

onMounted(() => {
	if (props.openDynamicModal) {
		fieldnames.value = props.openDynamicModal.dynamicContent;
		selectedDoctypeLabel.value = MainStore.doctype;
		if (props.table) {
			fieldnames.value = props.openDynamicModal.dynamicContent || [];
		}
	}
});

const size = {
	width: "75vw",
	height: "calc(94vh - 90px)",
	left: "6vw",
	top: "3vh",
};

const vFocus = {
	mounted: (el, binding) => {
		setTimeout(() => {
			el.focus();
			binding.modifiers.select && el.select();
		}, 500);
	},
};

const handleSidebarClick = async (fieldname) => {
	if (fieldname.length) {
		let meta = MainStore.metaFields.find((field) => field.fieldname == fieldname);
		selectedDoctypeLabel.value = meta.label || meta.fieldname;
		if (!meta.childfields) {
			await getMeta(meta.options, fieldname);
		}
	}
	previewRef.value.parentField = fieldname;
	previewRef.value.setParentField(fieldname);
};

const selectField = async (field, fieldtype) => {
	let isRemoved = false;
	fieldnames.value = fieldnames.value.filter((el) => {
		if (el.parentField == previewRef.value.parentField && el.fieldname == field.fieldname) {
			isRemoved = true;
			MainStore.dynamicData.splice(MainStore.dynamicData.indexOf(el), 1);
			if (previewRef.value.selectedEl && el == previewRef.value.selectedEl.field) {
				previewRef.value.setSelectedEl(null);
			}
			return false;
		}
		return true;
	});
	if (isRemoved) return;
	let index = fieldnames.value.length;
	let value = previewRef.value.parentField
					? await getValue(doctype.value, MainStore.docData[previewRef.value.parentField], field.fieldname)
					: props.table
					? typeof MainStore.docData[props.table.fieldname]?.[0][field.fieldname] != "undefined" ? frappe.format(
							MainStore.docData[props.table.fieldname][0][field.fieldname],
							{ fieldtype: field.fieldtype, options: field.options },
							{ inline: true },
							MainStore.docData
					  ) : `{{ ${field.fieldname} }}`
					: frappe.format(
							MainStore.docData[field.fieldname],
							{ fieldtype: field.fieldtype, options: field.options },
							{ inline: true },
							MainStore.docData
					  );
				if (!value) {
					if (["Image, Attach Image"].indexOf(field.fieldtype) != -1) {
						value = null;
					} else {
						switch (field.fieldname) {
							case "page":
								value = "0";
								break;
							case "topage":
								value = "999";
								break;
							case "date":
								value = frappe.datetime.now_date();
								break;
							case "time":
								value = frappe.datetime.now_time();
								break;
							default:
								value = `{{ ${previewRef.value.parentField ? previewRef.value.parentField + "." : ""}${
									field.fieldname
								} }}`;
						}
					}
				}
	let dynamicField = {
		doctype: doctype.value,
		parentField: previewRef.value.parentField,
		fieldname: field.fieldname,
		options: field.options,
		value,
		fieldtype,
		label: props.table ? field.label : `${field.label} :`,
		is_labelled: false,
		is_static: false,
		style: {},
		tableName: props.table?.fieldname,
		labelStyle: {},
		nextLine: !props.table,
	};
	if (previewRef.value.selectedEl) {
		index = previewRef.value.selectedEl.index + 1;
		fieldnames.value.splice(index, 0, dynamicField);
		MainStore.dynamicData.splice(index, 0, dynamicField);
	} else {
		fieldnames.value.push(dynamicField);
		MainStore.dynamicData.push(dynamicField);
	}
	previewRef.value.setSelectedEl({ index, field: dynamicField });
	if (props.table) {
		if (fieldnames.value?.length === 1) {
			previewRef.value.setLabel(fieldnames.value[0].label);
		} else if (fieldnames.value?.length === 0) {
			previewRef.value.setLabel("");
		}
	}
};

const primaryClick = (e) => {
	nextTick(() => {
		fieldnames.value.forEach((el) => {
			delete el["spanRef"];
			delete el["labelRef"];
		});
	});
	fieldnames.value.forEach((element) => {
		if (MainStore.dynamicData.indexOf(element) == -1) {
			MainStore.dynamicData.push(element);
		}
	});
	if (props.table) {
		props.openDynamicModal.label = "";
		fieldnames.value.length && (props.openDynamicModal.label = previewRef.value?.label || "");
		props.openDynamicModal.dynamicContent = [];
	}
	if (fieldnames.value.length) {
		props.openDynamicModal.dynamicContent = fieldnames.value;
	}
	props.openDynamicModal.selectedDynamicText = null;
	MainStore.openDynamicModal = null;
	MainStore.openTableColumnModal = null;
};
const cancelClick = () => {
	props.openDynamicModal.dynamicContent?.forEach((element) => {
		if (MainStore.dynamicData.indexOf(element) == -1) {
			MainStore.dynamicData.push(element);
		}
	});
	fieldnames.value.forEach((element) => {
		if (props.openDynamicModal.dynamicContent.indexOf(element) == -1) {
			MainStore.dynamicData.splice(MainStore.dynamicData.indexOf(element), 1);
		}
	});
	props.openDynamicModal.selectedDynamicText = null;
	MainStore.openDynamicModal = null;
	MainStore.openTableColumnModal = null;
};
</script>
<style scoped lang="scss">
small {
	font-size: 90%;
}
.dynamic-results {
	display: flex;
	flex: auto;
	max-height: max(calc(94vh - 318px), 150px);
	.searchbar {
		height: 26px;
		width: 100%;
		margin-bottom: 1rem;
		padding: 15px 0px 15px 25px;
		color: var(--text-color);
		background-color: var(--control-bg);
		font-size: var(--text-md);
		font-weight: 400;
		border: none;
		border-radius: var(--border-radius);
		&:focus {
			outline: none;
			box-shadow: none;
			border: none;
			outline: 1px solid var(--gray-300);
			outline-offset: -1px;
		}
	}
	.side-section {
		background-color: var(--bg-color);
		overflow: hidden;
		min-width: 16%;
		max-width: 16%;
		flex: 1;
		padding: 0px;
		border-radius: var(--border-radius-md);
		.searchbar {
			border-bottom: 1px solid var(--gray-200);
			outline-offset: -2px;
			border-radius: 0;
			&:focus {
				border-radius: var(--border-radius-md);
				border-bottom: 1px solid transparent;
			}
		}
		.sidebar {
			max-height: min(44vh, 650px);
			position: sticky;
			top: 0;
			overflow-y: auto;
			padding-bottom: var(--padding-md);
			padding-left: 0;
			list-style: none;
			&::-webkit-scrollbar {
				width: 5px;
				height: 5px;
			}
			&::-webkit-scrollbar-track,
			&::-webkit-scrollbar-corner {
				background: var(--bg-color);
			}
			&::-webkit-scrollbar-thumb {
				background: var(--gray-300);
				border-radius: 6px;
			}

			.sidebar-item {
				display: flex;
				align-items: center;
				font-size: 0.75rem;
				padding: 1px;
				width: 94%;
				margin: auto;
				margin-bottom: 3px;
				border-radius: var(--border-radius-md);
				overflow: hidden;
				cursor: pointer;

				span {
					padding: 8px 1px 8px 5px;
					margin-left: 3px;
				}
				&:hover,
				&.item-selected {
					border: 1px solid var(--gray-200);
					span {
						padding: 7px 0px 7px 4px;
					}
					background-color: white;
				}
			}
			.item-selected {
				border: 1px solid var(--gray-200);
				span {
					padding: 7px 0px 7px 4px;
				}
				background-color: white;
			}
		}
	}
	.main-section {
		flex: auto;
		padding: 0px 15px;
		max-height: min(50vh, 650px);
		overflow: auto;
		&::-webkit-scrollbar {
			width: 5px;
			height: 5px;
		}
		&::-webkit-scrollbar-track,
		&::-webkit-scrollbar-corner {
			background: white;
		}
		&::-webkit-scrollbar-thumb {
			background: var(--gray-300);
			border-radius: 6px;
		}
		.searchbar-sticky {
			margin-bottom: 0;
			position: sticky;
			top: 0;
			z-index: 1;
			background-color: var(--fg-color);
		}
		.container-main {
			padding: var(--padding-sm) 15px 0px var(--padding-sm);

			.fieldtype-title {
				display: flex;
				font-size: 1.25em;
				font-weight: bolder;
				padding: 15px 0px 6px 0px;
				margin-left: 0;
			}
			div:first-child > .fieldtype-title {
				padding-top: 6px;
			}
			.fields {
				display: flex;
				flex-wrap: wrap;
				.field {
					width: 100%;
					font-size: var(--text-md);
					flex: 33.3333333333%;
					max-width: 33.3333333333%;
					padding: var(--padding-sm) 0;
					border-bottom: 1px solid var(--border-color);

					.field-container {
						display: flex;
						align-items: center;
						padding: 8px 0;
						margin: 0px 8px;
						color: var(--text-light);
						.field-label {
							padding: 0px 15px;
							flex: 0 0 100%;
							max-width: 100%;
						}
						&:hover {
							border: 1px solid var(--success);
							padding: 7px 0px;
							border-radius: var(--border-radius);
							.field-label {
								padding: 0px 14px;
								flex: 0 0 100%;
								max-width: 100%;
							}
						}
					}
					.icon-show {
						display: none;
					}
					.field-selected {
						border: 1px solid var(--success);
						padding: 7px 0px;
						border-radius: var(--border-radius);
						.icon-show {
							display: unset;
							color: var(--success);
						}
						.field-label {
							font-weight: 600;
							padding: 0px 14px;
							display: flex;
							justify-content: space-between;
							align-items: center;
						}
					}
				}
			}
		}
	}
}
</style>
