<!--
  - @copyright Copyright (c) 2018 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<!-- If not in the rfcProps then we don't want to display it -->
	<component :is="componentInstance"
		v-if="propModel && propType !== 'unknown'"
		ref="component"
		:select-type.sync="selectType"
		:prop-model="propModel"
		:value.sync="value"
		:is-first-property="isFirstProperty"
		:property="property"
		:is-last-property="isLastProperty"
		:class="{'property--last': isLastProperty}"
		:local-contact="localContact"
		:prop-name="propName"
		:prop-type="propType"
		:options="sortedModelOptions"
		:is-read-only="isReadOnly"
		@delete="deleteProp"
		@update="updateContact" />
</template>

<script>
import { Property } from 'ical.js'
import rfcProps from '../../models/rfcProps'
import Contact from '../../models/contact'

import PropertyText from '../Properties/PropertyText'
import PropertyMultipleText from '../Properties/PropertyMultipleText'
import PropertyDateTime from '../Properties/PropertyDateTime'
import PropertySelect from '../Properties/PropertySelect'

export default {
	name: 'ContactDetailsProperty',

	props: {
		property: {
			type: Property,
			default: true,
		},
		sortedProperties: {
			type: Array,
			default() {
				return []
			},
		},
		index: {
			type: Number,
			default: 0,
		},
		contact: {
			type: Contact,
			default: null,
		},
		localContact: {
			type: Contact,
			default: null,
		},
		/**
		 * This is needed so that we can update
		 * the contact within the rfcProps actions
		 */
		updateContact: {
			type: Function,
			default: () => {},
		},
	},

	computed: {
		// dynamically load component based on property type
		componentInstance() {
			// dynamic matching
			if (this.property.isMultiValue && this.propType === 'text') {
				return PropertyMultipleText
			} else if (this.propType && ['date-and-or-time', 'date-time', 'time', 'date'].indexOf(this.propType) > -1) {
				return PropertyDateTime
			} else if (this.propType && this.propType === 'select') {
				return PropertySelect
			} else if (this.propType && this.propType !== 'unknown') {
				return PropertyText
			}
			return PropertyText
		},

		// rfc properties list
		properties() {
			return rfcProps.properties
		},
		fieldOrder() {
			return rfcProps.fieldOrder
		},

		// is this the first property of its kind
		isFirstProperty() {
			if (this.index > 0) {
				return this.sortedProperties[this.index - 1].name.split('.').pop() !== this.propName
			}
			return true
		},
		// is this the last property of its kind
		isLastProperty() {
			// array starts at 0, length starts at 1
			if (this.index < this.sortedProperties.length - 1) {
				return this.sortedProperties[this.index + 1].name.split('.').pop() !== this.propName
			}
			return true
		},
		isReadOnly() {
			if (this.contact.addressbook) {
				return this.contact.addressbook.readOnly
			}
			return false
		},

		/**
		 * Return the type of the prop e.g. FN
		 *
		 * @returns {string}
		 */
		propName() {
			// ! is this a ITEMXX.XXX property??
			if (this.propGroup[1]) {
				return this.propGroup[1]
			}

			return this.property.name
		},
		/**
		 * Return the type or property
		 *
		 * @see src/models/rfcProps
		 * @returns {string}
		 */
		propType() {
			// if we have a force type set, use it!
			if (this.propModel && this.propModel.force) {
				return this.propModel.force
			}

			return this.property.getDefaultType()
		},

		/**
		 * RFC template matching this property
		 *
		 * @see src/models/rfcProps
		 * @returns {Object}
		 */
		propModel() {
			return this.properties[this.propName]
		},

		/**
		 * Remove duplicate name amongst options
		 * but make sure to include the selected one
		 * in the final list
		 *
		 * @returns {Object[]}
		 */
		sortedModelOptions() {
			if (this.propModel.options) {
				return this.propModel.options.reduce((list, option) => {
					if (!list.find(search => search.name === option.name)) {
						list.push(option)
					}
					return list
				}, this.selectType ? [this.selectType] : [])
			}
			return []
		},

		/**
		 * Return the id and type of a property group
		 * e.g ITEMXX.tel => ['ITEMXX', 'tel']
		 *
		 * @returns {Array}
		 */
		propGroup() {
			return this.property.name.split('.')
		},

		/**
		 * Return the associated X-ABLABEL if any
		 *
		 * @returns {Property}
		 */
		propLabel() {
			return this.localContact.vCard.getFirstProperty(`${this.propGroup[0]}.x-ablabel`)
		},

		/**
		 * Returns the closest match to the selected type
		 * or return the default selected as a new object if
		 * none exists
		 *
		 * @returns Object|null
		 */
		selectType: {
			get() {
				// ! if ABLABEL is present, this is a priority
				if (this.propLabel) {
					return {
						id: this.propLabel.name,
						name: this.propLabel.getFirstValue(),
					}
				}
				if (this.propModel && this.propModel.options && this.type) {

					const selectedType = this.type
						// vcard 3.0 save pref alongside TYPE
						.filter(type => type !== 'pref')
						// we only use uppercase strings
						.map(str => str.toUpperCase())

					// Compare array and score them by how many matches they have to the selected type
					// sorting directly is cleaner but slower
					// https://jsperf.com/array-map-and-intersection-perf
					const matchingTypes = this.propModel.options
						.map(type => {
							return {
								type,
								// "WORK,HOME" => ['WORK', 'HOME']
								score: type.id.split(',').filter(value => selectedType.indexOf(value) !== -1).length,
							}
						})

					// Sort by score, filtering out the null score and selecting the first match
					const matchingType = matchingTypes
						.sort((a, b) => b.score - a.score)
						.filter(type => type.score > 0)[0]

					if (matchingType) {
						return matchingType.type
					}
				}
				if (this.type) {
					// vcard 3.0 save pref alongside TYPE
					const selectedType = this.type
						.filter(type => type !== 'pref')
						.join(',')
					if (selectedType.trim() !== '') {
						return {
							id: selectedType,
							name: selectedType,
						}
					}
				}
				return null
			},
			set(data) {
				// if a custom label exists and this is the one we selected
				if (this.propLabel && data.id === this.propLabel.name) {
					this.propLabel.setValue(data.name)
					// only one can coexist
					this.type = []
				} else {
					// ical.js take types as arrays
					this.type = data.id.split(',')
					// only one can coexist
					this.localContact.vCard.removeProperty(`${this.propGroup[0]}.x-ablabel`)

					// checking if there is any other property in this group
					const groups = this.localContact.jCal[1]
						.map(prop => prop[0])
						.filter(name => name.startsWith(`${this.propGroup[0]}.`))
					if (groups.length === 1) {
						// then this prop is the latest of its group
						// -> converting back to simple prop
						this.property.jCal[0] = this.propGroup[1]
					}
				}
				this.updateContact()
			},

		},

		// property value(s)
		value: {
			get() {
				if (this.property.isMultiValue) {
					// differences between values types :x;x;x;x;x and x,x,x,x,x
					return this.property.isStructuredValue
						? this.property.getValues()[0]
						: this.property.getValues()
				}
				return this.property.getFirstValue()
			},
			set(data) {
				if (this.property.isMultiValue) {
					// differences between values types :x;x;x;x;x and x,x,x,x,x
					this.property.isStructuredValue
						? this.property.setValues([data])
						: this.property.setValues(data)
				} else {
					this.property.setValue(data)
				}
				this.updateContact()
			},
		},

		// property meta type
		type: {
			get() {
				const type = this.property.getParameter('type')
				// ensure we have an array
				if (type) {
					return Array.isArray(type) ? type : [type]
				}
				return null
			},
			set(data) {
				this.property.setParameter('type', data)
			},
		},

		// property meta pref
		pref: {
			get() {
				return this.property.getParameter('pref')
			},
			set(data) {
				this.property.setParameter('pref', data)
			},
		},
	},

	methods: {
		/**
		 * Delete this property
		 */
		deleteProp() {
			this.localContact.vCard.removeProperty(this.property)
			this.updateContact()
		},
	},
}
</script>
