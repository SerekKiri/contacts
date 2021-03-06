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
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div id="contact-details" class="app-content-details">

		<!-- nothing selected or contact not found -->
		<div v-if="!contact && !loading" id="emptycontent">
			<div class="icon-contacts" />
			<h2>{{ t('contacts', 'No contact selected') }}</h2>
			<p>{{ t('contacts', 'Select a contact on the list to begin') }}</p>
		</div>

		<!-- loading -->
		<div v-else-if="!contact && loading" id="emptycontent">
			<div class="icon-contacts" />
			<h2>{{ t('contacts', 'Loading') }}</h2>
		</div>

		<template v-else>
			<!-- contact header -->
			<header :style="{ 'backgroundColor': colorAvatar }">

				<!-- avatar and upload photo -->
				<contact-avatar :contact="contact" />
				<!-- QUESTION: is it better to pass contact as a prop or get it from the store inside
				contact-avatar ?  :avatar="contact.photo"-->

				<!-- fullname, org, title -->
				<div id="contact-header-infos">
					<h2>
						<input id="contact-fullname" v-model="contact.fullName" :disabled="!contact.addressbook.readOnly"
							:placeholder="t('contacts', 'Name')" type="text" autocomplete="off"
							autocorrect="off" spellcheck="false" name="fullname"
							value="" @input="debounceUpdateContact">
					</h2>
					<div id="details-org-container">
						<input id="contact-org" v-model="contact.org" :disabled="!contact.addressbook.readOnly"
							:placeholder="t('contacts', 'Company')" type="text" autocomplete="off"
							autocorrect="off" spellcheck="false" name="org"
							value="" @input="debounceUpdateContact">
						<input id="contact-title" v-model="contact.title" :disabled="!contact.addressbook.readOnly"
							:placeholder="t('contacts', 'Title')" type="text" autocomplete="off"
							autocorrect="off" spellcheck="false" name="title"
							value="" @input="debounceUpdateContact">
					</div>
				</div>

				<!-- actions -->
				<div id="contact-header-actions">
					<div v-tooltip.bottom="warning" :class="{'icon-loading-small': loadingUpdate, 'menu-icon--pulse icon-error-white': warning}" class="menu-icon" />
					<div v-tooltip="{
							content: conflict,
							show: true,
							trigger: 'manual',
						}" v-if="conflict" class="menu-icon menu-icon--pulse icon-history-white"
						@click="refreshContact" />
					<div class="menu-icon">
						<div v-click-outside="closeMenu" class="icon-more-white" @click="toggleMenu" />
						<div :class="{ 'open': openedMenu }" class="popovermenu">
							<popover-menu :menu="contactActions" />
						</div>
					</div>
				</div>
			</header>

			<!-- contact details loading -->
			<section v-if="loadingData" class="icon-loading contact-details" />

			<!-- contact details -->
			<section v-else class="contact-details">

				<!-- properties iteration -->
				<!-- using contact.key in the key and index as key to avoid conflicts between similar data and exact key -->
				<contact-property v-for="(property, index) in sortedProperties" :key="index+contact.key" :index="index"
					:sorted-properties="sortedProperties" :property="property" :contact="contact"
					@updatedcontact="updateContact" />

				<!-- addressbook change select - no last property because class is not applied here-->
				<property-select :prop-model="addressbookModel" :value.sync="addressbook" :is-first-property="true"
					:is-last-property="false" class="property--addressbooks" />

				<!-- new property select -->
				<add-new-prop :contact="contact" />
			</section>
		</template>
	</div>
</template>

<script>
import { PopoverMenu } from 'nextcloud-vue'
import ClickOutside from 'vue-click-outside'
import Vue from 'vue'
import VTooltip from 'v-tooltip'
import debounce from 'debounce'

import Contact from '../models/contact'
import rfcProps from '../models/rfcProps.js'

import ContactProperty from './ContactDetails/ContactDetailsProperty'
import AddNewProp from './ContactDetails/ContactDetailsAddNewProp'
import PropertySelect from './Properties/PropertySelect'
import PropertyGroups from './Properties/PropertyGroups'
import ContactAvatar from './ContactDetails/ContactDetailsAvatar'

Vue.use(VTooltip)

export default {
	name: 'ContactDetails',

	components: {
		PopoverMenu,
		ContactProperty,
		PropertySelect,
		PropertyGroups,
		AddNewProp,
		ContactAvatar
	},

	directives: {
		ClickOutside
	},

	props: {
		loading: {
			type: Boolean,
			default: true
		},
		uid: {
			type: String,
			default: undefined
		}
	},

	data() {
		return {
			/**
			 * Local off-store clone of the selected contact for edition
			 * because we can't edit contacts data outside the store.
			 * Every change will be dispatched and updated on the real
			 * store contact after a debounce.
			 */
			localContact: undefined,
			loadingData: true,
			loadingUpdate: false,
			openedMenu: false
		}
	},

	computed: {

		/**
		 * Warning message
		 *
		 * @returns {string|undefined}
		 */
		warning() {
			if (!this.contact.dav) {
				return t('contacts', 'This contact is not yet synced. Edit it to trigger a change.')
			}
		},

		/**
		 * Conflict message
		 *
		 * @returns {string|undefined}
		 */
		conflict() {
			if (this.contact.conflict) {
				return t('contacts', 'The contact you were trying to edit has changed. Please manually refresh the contact. Any further edits will be discarded.')
			}
		},

		/**
		 * Contact color based on uid
		 *
		 * @returns {string}
		 */
		colorAvatar() {
			try {
				let color = this.contact.uid.toRgb()
				return `rgb(${color.r}, ${color.g}, ${color.b})`
			} catch (e) {
				return 'grey'
			}
		},

		/**
		 * Header actions for the contact
		 *
		 * @returns {Array}
		 */
		contactActions() {
			let actions = [
				{
					icon: 'icon-download',
					text: t('contacts', 'Download'),
					href: this.contact.url
				}
			]
			if (this.contact.addressbook.readOnly) {
				actions.push({
					icon: 'icon-delete',
					text: t('contacts', 'Delete'),
					action: this.deleteContact
				})
			}

			return actions
		},

		/**
		 * Contact properties copied and sorted by rfcProps.fieldOrder
		 *
		 * @returns {Array}
		 */
		sortedProperties() {
			return this.contact.properties.slice(0).sort((a, b) => {
				return (
					rfcProps.fieldOrder.indexOf(a.name) - rfcProps.fieldOrder.indexOf(b.name)
				)
			})
		},

		/**
		 * Fake model to use the propertySelect component
		 *
		 * @returns {Object}
		 */
		addressbookModel() {
			return {
				readableName: t('contacts', 'Addressbook'),
				icon: 'icon-addressbook',
				options: this.addressbooksOptions
			}
		},

		/**
		 * Usable addressbook object linked to the local contact
		 *
		 * @param {string} [addressbookId] set the addressbook id
		 * @returns {string}
		 */
		addressbook: {
			get: function() {
				return this.contact.addressbook.id
			},
			set: function(addressbookId) {
				this.moveContactToAddressbook(addressbookId)
			}
		},

		/**
		 * Store getters filtered and mapped to usable object
		 *
		 * @returns {Array}
		 */
		addressbooksOptions() {
			return this.addressbooks
				.filter(addressbook => addressbook.readOnly)
				.map(addressbook => {
					return {
						id: addressbook.id,
						name: addressbook.displayName
					}
				})
		},

		// store getter
		addressbooks() {
			return this.$store.getters.getAddressbooks
		},
		contact() {
			return this.$store.getters.getContact(this.uid)
		}
	},

	watch: {
		contact: function() {
			if (this.uid) {
				this.selectContact(this.uid)
			}
		}
	},

	beforeMount() {
		// load the desired data if we already selected a contact
		if (this.uid) {
			this.selectContact(this.uid)
		}
	},

	methods: {
		/**
		 * Executed on the 'updatedcontact' event
		 * Send the local clone of contact to the store
		 */
		updateContact() {
			this.loadingUpdate = true
			this.$store.dispatch('updateContact', this.contact)
				.then(() => {
					this.loadingUpdate = false
				})
		},

		/**
		 * Debounce the contact update for the header props
		 * photo, fn, org, title
		 */
		debounceUpdateContact: debounce(function(e) {
			this.updateContact()
		}, 500),

		// menu handling
		closeMenu() {
			this.openedMenu = false
		},
		toggleMenu() {
			this.openedMenu = !this.openedMenu
		},

		/**
		 * Select a contac, and update the localContact
		 * Fetch updated data if necessary
		 * Scroll to the selected contact if exists
		 *
		 * @param {string} uid the contact uid
		 */
		selectContact(uid) {
			// local version of the contact
			this.loadingData = true
			let contact = this.$store.getters.getContact(uid)

			if (contact) {
				// if contact exists AND if exists on server
				if (contact.dav) {
					this.$store.dispatch('fetchFullContact', { contact })
						.then(() => {
							// create empty contact and copy inner data
							let localContact = new Contact(
								'BEGIN:VCARD\nUID:' + contact.uid + '\nEND:VCARD',
								contact.addressbook
							)
							localContact.updateContact(contact.jCal)
							this.localContact = localContact
							this.loadingData = false
						})
						.catch((error) => {
							OC.Notification.showTemporary(t('contacts', 'The contact doesn\'t exists anymore on the server.'))
							console.error(error)
							// trigger a local deletion from the store only
							this.$store.dispatch('deleteContact', { contact: this.contact, dav: false })
						})
				} else {
					// create empty contact and copy inner data
					// wait for an update to really push the contact on the server!
					this.localContact = new Contact(
						'BEGIN:VCARD\nUID:' + contact.uid + '\nEND:VCARD',
						contact.addressbook
					)
					this.loadingData = false
				}

				// scroll to selected contact if any
				let list = document.getElementById('contacts-list')
				let item = document.querySelector('#' + btoa(contact.key).slice(0, -2))
				let isAbove = list.scrollTop > item.offsetTop
				let isUnder = item.offsetTop + item.offsetHeight > list.scrollTop + list.offsetHeight
				// check if contact outside visible list area
				if (item && (isAbove || isUnder)) {
					list.scrollTo(0, item.offsetTop - item.offsetHeight / 2)
				}
			}
		},

		/**
		 * Dispatch contact deletion request
		 */
		deleteContact() {
			this.$store.dispatch('deleteContact', { contact: this.contact })
		},

		/**
		 * Move contact to the specified addressbook
		 *
		 * @param {string} addressbookId the desired addressbook ID
		 */
		moveContactToAddressbook(addressbookId) {
			let addressbook = this.addressbooks.find(search => search.id === addressbookId)
			this.loadingUpdate = true
			// TODO Properly implement the MOVE request
			if (addressbook) {
				this.$store.dispatch('moveContactToAddressbook', {
					// we need to use the store contact, not the local contact
					// using this.contact and not this.localContact
					contact: this.contact,
					addressbook
				}).then(() => {
					this.updateContact()
					this.loadingUpdate = false
				})
			}
		},

		/**
		 * Refresh the data of a contact
		 */
		refreshContact() {
			this.$store.dispatch('fetchFullContact', { contact: this.contact, etag: this.conflict })
				.then(() => {
					this.contact.conflict = false
				})
		}
	}
}
</script>
