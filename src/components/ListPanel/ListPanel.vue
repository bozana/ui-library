<template>
	<div class="pkpListPanel" :class="classStatus">
		<div class="pkpListPanel__header">
			<div class="pkpListPanel__title">{{ i18n.title }}</div>
		</div>
		<div class="pkpListPanel__body">
			<ul class="pkpListPanel__items" aria-live="polite">
				<list-panel-item
					v-for="item in collection.items"
					:key="item.id"
					:item="item"
				/>
			</ul>
		</div>
	</div>
</template>

<script>
import ListPanelItem from './ListPanelItem.vue';

export default {
	name: 'ListPanel',
	components: {
		ListPanelItem,
	},
	data: function () {
		return {
			id: '',
			collection: {
				items: [],
				maxItems: null,
			},
			filterParams: {},
			searchPhrase: '',
			isLoading: false,
			isSearching: false,
			isOrdering: false,
			isFilterVisible: false,
			count: 20,
			offset: 0,
			apiPath: '',
			getParams: {},
			i18n: {},
			lazyLoad: false,
			_lastGetRequest: null,
		};
	},
	computed: {
		classStatus: function () {
			return { '--isLoading': this.isLoading };
		},
		itemCount: function () {
			return this.collection.items.length;
		},
		canLoadMore: function () {
			return typeof this.collection.maxItems !== 'undefined' && this.collection.maxItems > this.itemCount;
		},

		/**
		 * Options for the draggable component
		 *
		 * @see https://github.com/SortableJS/Vue.Draggable
		 */
		draggableOptions: function () {
			return {
				disabled: !this.isOrdering,
			};
		},
	},
	methods: {

		/**
		 * Set the search phrase
		 *
		 * @param string val The new search phrase
		 */
		setSearchPhrase: function (val) {
			this.searchPhrase = val;
		},

		/**
		 * Get items for the list. This ListPanel must have a defined
		 * `get` route to execute this method.
		 *
		 * @param string statusIndicator The key for the data which should be
		 *  toggled while this action is being performed. Default: `isLoading`
		 *  corresponds with this.isLoading. The data referenced must be a bool
		 * @param string handleResponse How to handle the response. `append` to
		 *  add to the collection. Default: null will replace the collection.
		 */
		get: function (statusIndicator, handleResponse) {

			if (typeof statusIndicator === 'undefined') {
				statusIndicator = 'isLoading';
			}

			this[statusIndicator] = true;

			var self = this;

			// Address issues with multiple async get requests. Store an ID for the
			// most recent get request. When we receive the response, we
			// can check that the response matches the most recent get request, and
			// discard responses that are outdated.
			this._latestGetRequest = $.pkp.classes.Helper.uuid();

			$.ajax({
				url: $.pkp.app.apiBaseUrl + '/' + this.apiPath,
				type: 'GET',
				data: _.extend(
					{},
					this.getParams,
					this.filterParams,
					{
						searchPhrase: this.searchPhrase,
						count: this.count,
						offset: this.offset,
					},
				),
				_uuid: this._latestGetRequest,
				error: function (r) {

					// Only process latest request response
					if (self._latestGetRequest !== this._uuid) {
						return;
					}
					self.ajaxErrorCallback(r);
				},
				success: function (r) {

					// Only process latest request response
					if (self._latestGetRequest !== this._uuid) {
						return;
					}

					if (handleResponse === 'append') {
						var existingItemIds = _.pluck(self.items, 'id');
						_.each(r.items, function (item) {
							if (existingItemIds.indexOf(item.id) < 0) {
								self.collection.items.push(item);
							}
						});
						self.collection.maxItems = r.maxItems;
					} else {
						self.collection = r;
					}
				},
				complete: function (r) {

					// Only process latest request response
					if (self._latestGetRequest !== this._uuid) {
						return;
					}

					self[statusIndicator] = false;
				},
			});
		},

		/**
		 * Load more items in the list
		 */
		loadMore: function () {
			this.offset = this.collection.items.length;
			this.get('isLoading', 'append');
		},

		/**
		 * Toggle filter visibility
		 */
		toggleFilter: function () {
			this.isFilterVisible = !this.isFilterVisible;
		},

		/**
		 * Update filter parameters
		 */
		updateFilter: function (params) {
			this.filterParams = params;
		},

		/**
		 * Toggle the ordering status
		 */
		toggleOrdering: function () {
			if (this.isOrdering) {
				this.setItemOrderSequence();
			}
			this.isOrdering = !this.isOrdering;
			if (this.isOrdering) {
				this.$nextTick(function () {
					var helpEl = this.$el.querySelector('.pkpListPanel__notice');
					if (helpEl) {
						helpEl.focus();
					}
				});
			}
		},

		/**
		 * Move an item up in the list
		 */
		itemOrderUp: function (data) {
			var index = _.findIndex(this.collection.items, function (item) { return item.id == data.id; });
			if (index === 0) {
				return;
			}
			this.collection.items.splice(index - 1, 0, this.collection.items.splice(index, 1)[0]);
			this.itemOrderResetFocus(data.id, 'up');
		},

		/**
		 * Move an item down in the list
		 */
		itemOrderDown: function (data) {
			var index = _.findIndex(this.collection.items, function (item) { return item.id == data.id; });
			if (index === this.collection.items.length - 1) {
				return;
			}
			this.collection.items.splice(index + 1, 0, this.collection.items.splice(index, 1)[0]);
			this.itemOrderResetFocus(data.id, 'down');
		},

		/**
		 * Move focus to up/down button for item that was just moved
		 *
		 * When using the up/down arrows, the focus stays on the button in
		 * the position which was clicked. This function ensures the focus
		 * travels with the item that's been moved.
		 *
		 * For the same reason, we have to do a manual look up on the child
		 * component by id. Vue.js's optimization code swaps out the items
		 * without resetting the components, so any callback is fired on the
		 * item which is in the position of the item that was just moved. In
		 * other words, under-the-hood Vue.js moves the data around but leaves
		 * the components in place, so we have to manually find the component
		 * where the moved item is and set focus there.
		 *
		 * @param itemId int The id of the item to set focus in
		 * @param direction string Set focus on the 'up' or 'down' btn
		 */
		itemOrderResetFocus: function (itemId, direction) {

			// Wait until the components have been redrawn before setting focus
			this.$nextTick(function () {
				_.each(this.$children, function (child) {
					// If the list items are nested inside a draggable,
					// search in that component's children.
					if (child.$options._componentTag === 'draggable') {
						var listItem = _.findWhere(child.$children, {id: itemId});
						_.each(listItem.$children, function (itemChild) {
							if (itemChild.$options._componentTag === 'list-panel-item-orderer') {
								itemChild.setFocus(direction);
							}
						});
						return false;
					} else if (child.id === itemId) {
						_.each(child.$children, function (itemChild) {
							if (itemChild.$options._componentTag === 'list-panel-item-orderer') {
								itemChild.setFocus(direction);
							}
						});
						return false;
					}
				}, this);
			});
		},

		/**
		 * Update the order sequence property for items in this list based on
		 * the new order of items
		 */
		setItemOrderSequence: function (prop) {
			prop = prop || 'seq'; // default sequence property in item models

			_.each(this.collection.items, function (item, i) {
				item[prop] = i;
			});
		},

		/**
		 * Cancel changes made by ordering items
		 */
		cancelOrdering: function () {
			this.isOrdering = false;
			this.offset = 0;
			this.get();
		},
	},
	mounted: function () {
		/**
		 * Perform a search whenever the searchPhrase is updated
		 */
		this.$watch('searchPhrase', function (newVal, oldVal) {
			if (newVal === oldVal) {
				return;
			}
			this.offset = 0;
			this.get('isSearching');
		});

		/**
		 * Update list whenever a filter is applied
		 */
		this.$watch('filterParams', function (newVal, oldVal) {
			if (newVal === oldVal) {
				return;
			}
			this.offset = 0;
			this.get('isLoading');
		});

		/**
		 * Load a collection into the component once the page is loaded if a
		 * lazyLoad is requested.
		 */
		if (this.lazyLoad) {
			if (document.readyState === 'complete') {
				this.get();
			} else {
				var self = this;
				$(function () {
					self.get();
				});
			}
		}
	},
};
</script>

<style lang="less">
@import '../../styles/_config';

.pkpListPanel {
	position: relative;
	max-width: 900px;
	box-shadow: 0 1px 1px rgba(0,0,0,0.2);
	border-radius: @radius;

	&.--isLoading {

		.pkpListPanel__title:after {
			&:extend(.pkp_spinner:after);
			position: absolute;
			top: 50%;
			margin-top: -10px;
			margin-left: 1em;
		}
	}
}

.pkpListPanel__header {
	.pkp_helpers_clear;
	background: @bg-light;
	padding: 0 0.5em 0.5em 1em;
}

.pkpListPanel__title {
	position: relative;
	margin-top: 0.5em;
	font-weight: @bold;
	float: left;
	max-width: 30em;
}

.pkpListPanel__actions {
	float: right;
	margin: 0.5em 0 0;
	padding: 0;
	max-width: 30em;

	li {
		display: inline-block;
	}
}

.pkpListPanel__body {
	.pkp_helpers_clear;
	position: relative;
}

.pkpListPanel__filter.--isVisible {
	float: left;
	position: relative;
	left: auto;
	width: 25%;

	+ .pkpListPanel__content  {
		float: right;
		width: 75%;
	}

	.pkpListPanel__filterHeader,
	.pkpListPanel__filterOptions {
		position: relative;
		left: 0;
		opacity: 1;
		width: 100%;
		transition: opacity 0.2s ease-in-out 0.2s, left 0s ease-in-out 0.2s, width 0s ease-in-out 0.2s;
	}
}

.pkpListPanel__columnLabels {
	position: relative;
	display: block;
	min-height: @base * 3;
	background: @lift;
	border-bottom: @grid-border;
	font-size: @font-tiny;
	line-height: @line-tiny;
}

.pkpListPanel__items {
	margin: 0;
	padding: 0;
}
.pkpListPanel__footer {
	.pkp_helpers_clear();
	position: relative;
	padding: @half @base;
	font-size: @font-tiny;
	line-height: @line-base;
	border-top: @grid-border;
}

.pkpListPanel.--isOrdering {

	// Hide other items on screen
	.pkpListPanel__actions > li:not(.pkpListPanel__orderToggle):not(.pkpListPanel__orderToggleCancel),
	.pkpListPanel__search {
		display: none;
	}
}

.pkpListPanelItem__ordererDragDrop,
.pkpListPanelItem__ordererUp,
.pkpListPanelItem__ordererDown {
	position: absolute;
	top: 0;
	left: 0;
	width: 4em;
	height: 100%;
	text-align: center;
	background: transparent;
	border: none;

	&:focus {
		outline: 0;
	}
}

.pkpListPanelItem__ordererDragDrop {
	color: @primary;
	border-right: @grid-border;
	cursor: move;
}

.pkpListPanelItem__ordererUp,
.pkpListPanelItem__ordererDown {
	left: auto;
	right: 0;
	color: @text-light;
	border-left: @grid-border;

	&:hover {
		color: @primary;
		box-shadow: 0 2px 0 @primary;
	}

	&:focus {
		background: @bg-light;
	}
}

.pkpListPanelItem__ordererUp {
	right: 4em;
}

.pkpListPanel__notice {
	padding: 1em;
	border-bottom: @grid-border;
	font-size: @font-sml;
	line-height: @line-sml;

	&:focus {
		box-shadow: -2px 0 0 @primary;
		outline: 0;
	}
}

.pkpListPanelItem__selectItem {
	position: absolute;
	top: 0;
	width: 5em;
	height: 100%;
	background: transparent;
	border: none;
	border-right: @grid-border;
}
</style>