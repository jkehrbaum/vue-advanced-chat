<template>
	<div>
		<div v-if="showDate" class="vac-card-info vac-card-date">
			{{ message.date }}
		</div>

		<div v-if="newMessage._id === message._id" class="vac-line-new">
			{{ textMessages.NEW_MESSAGES }}
		</div>

		<div v-if="message.system" class="vac-card-info vac-card-system">
			{{ message.content }}
		</div>

		<div
			v-else
			:id="message._id"
			class="vac-message-box"
			:class="{ 'vac-offset-current': message.sender_id === currentUserId }"
		>
			<slot name="message" v-bind="{ message }">
				<div
					class="vac-message-container"
					:class="{
						'vac-message-container-offset': messageOffset
					}"
				>
					<div
						class="vac-message-card"
						:class="{
							'vac-message-highlight': isMessageHover,
							'vac-message-current': message.sender_id === currentUserId,
							'vac-message-deleted': message.deleted
						}"
						@mouseover="onHoverMessage(message)"
						@mouseleave="onLeaveMessage"
					>
						<div
							v-if="roomUsers.length > 2 && message.sender_id !== currentUserId"
							class="vac-text-username"
							:class="{
								'vac-username-reply': !message.deleted && message.replyMessage
							}"
						>
							<span>{{ message.username }}</span>
						</div>

						<message-reply
							v-if="!message.deleted && message.replyMessage"
							:message="message"
							:room-users="roomUsers"
						></message-reply>

						<div v-if="message.deleted">
							<slot name="deleted-icon">
								<svg-icon name="deleted" class="vac-icon-deleted" />
							</slot>
							<span>{{ textMessages.MESSAGE_DELETED }}</span>
						</div>

						<format-message
							v-else-if="!message.file"
							:content="message.content"
							:users="roomUsers"
							:text-formatting="textFormatting"
							@open-user-tag="openUserTag"
						>
							<template v-slot:deleted-icon="data">
								<slot name="deleted-icon" v-bind="data"></slot>
							</template>
						</format-message>

						<message-image
							v-else-if="isImage"
							:current-user-id="currentUserId"
							:message="message"
							:room-users="roomUsers"
							:text-formatting="textFormatting"
							:image-hover="imageHover"
							@open-file="openFile"
						>
							<template
								v-for="(index, name) in $scopedSlots"
								v-slot:[name]="data"
							>
								<slot :name="name" v-bind="data"></slot>
							</template>
						</message-image>

						<div v-else-if="isVideo" class="vac-video-container">
							<video width="100%" height="100%" controls>
								<source :src="message.file.url" />
							</video>
						</div>

						<div v-else-if="message.file.audio" class="vac-audio-message">
							<div id="vac-audio-player">
								<audio controls v-if="message.file.audio">
									<source :src="message.file.url" />
								</audio>
							</div>
						</div>

						<div v-else class="vac-file-message">
							<div
								class="vac-svg-button vac-icon-file"
								@click.stop="openFile('download')"
							>
								<slot name="document-icon">
									<svg-icon name="document" />
								</slot>
							</div>
							<span>{{ message.content }}</span>
						</div>

						<div class="vac-text-timestamp">
							<div
								class="vac-icon-edited"
								v-if="message.edited && !message.deleted"
							>
								<slot name="pencil-icon">
									<svg-icon name="pencil" />
								</slot>
							</div>
							<span>{{ message.timestamp }}</span>
							<span v-if="isCheckmarkVisible">
								<slot name="checkmark-icon" v-bind="{ message }">
									<svg-icon
										:name="
											message.distributed ? 'double-checkmark' : 'checkmark'
										"
										:param="message.seen ? 'seen' : ''"
										class="vac-icon-check"
									></svg-icon>
								</slot>
							</span>
						</div>

						<message-actions
							:current-user-id="currentUserId"
							:message="message"
							:message-actions="messageActions"
							:room-footer-ref="roomFooterRef"
							:show-reaction-emojis="showReactionEmojis"
							:hide-options="hideOptions"
							:message-hover="messageHover"
							:hover-message-id="hoverMessageId"
							@hide-options="$emit('hide-options', false)"
							@update-message-hover="messageHover = $event"
							@update-options-opened="optionsOpened = $event"
							@update-emoji-opened="emojiOpened = $event"
							@message-action-handler="messageActionHandler"
							@send-message-reaction="sendMessageReaction($event)"
						>
							<template
								v-for="(index, name) in $scopedSlots"
								v-slot:[name]="data"
							>
								<slot :name="name" v-bind="data"></slot>
							</template>
						</message-actions>
					</div>

					<message-reactions
						:current-user-id="currentUserId"
						:message="message"
						:emojis-list="emojisList"
						@send-message-reaction="sendMessageReaction($event)"
					></message-reactions>
				</div>
			</slot>
		</div>
	</div>
</template>

<script>
import SvgIcon from '../../components/SvgIcon'
import FormatMessage from '../../components/FormatMessage'

import MessageReply from './MessageReply'
import MessageImage from './MessageImage'
import MessageActions from './MessageActions'
import MessageReactions from './MessageReactions'

const { isImageFile } = require('../../utils/mediaFile')

export default {
	name: 'message',
	components: {
		SvgIcon,
		FormatMessage,
		MessageReply,
		MessageImage,
		MessageActions,
		MessageReactions
	},

	props: {
		currentUserId: { type: [String, Number], required: true },
		textMessages: { type: Object, required: true },
		index: { type: Number, required: true },
		message: { type: Object, required: true },
		messages: { type: Array, required: true },
		editedMessage: { type: Object, required: true },
		roomUsers: { type: Array, default: () => [] },
		messageActions: { type: Array, required: true },
		roomFooterRef: { type: HTMLDivElement },
		newMessages: { type: Array },
		showReactionEmojis: { type: Boolean, required: true },
		showNewMessagesDivider: { type: Boolean, required: true },
		textFormatting: { type: Boolean, required: true },
		emojisList: { type: Object, required: true },
		hideOptions: { type: Boolean, required: true }
	},

	data() {
		return {
			hoverMessageId: null,
			imageHover: false,
			messageHover: false,
			optionsOpened: false,
			emojiOpened: false,
			messageReaction: '',
			newMessage: {}
		}
	},

	mounted() {
		if (!this.message.seen && this.message.sender_id !== this.currentUserId) {
			this.$emit('add-new-message', {
				_id: this.message._id,
				index: this.index
			})
		}
	},

	watch: {
		newMessages(val) {
			if (!val.length || !this.showNewMessagesDivider) return
			this.newMessage = val.reduce((res, obj) =>
				obj.index < res.index ? obj : res
			)
		}
	},

	computed: {
		showDate() {
			return (
				this.index > 0 &&
				this.message.date !== this.messages[this.index - 1].date
			)
		},
		messageOffset() {
			return (
				this.index > 0 &&
				this.message.sender_id !== this.messages[this.index - 1].sender_id
			)
		},
		isMessageHover() {
			return (
				this.editedMessage._id === this.message._id ||
				this.hoverMessageId === this.message._id
			)
		},
		isImage() {
			return isImageFile(this.message.file)
		},
		isVideo() {
			return this.checkVideoType(this.message.file)
		},
		isCheckmarkVisible() {
			return (
				this.message.sender_id === this.currentUserId &&
				!this.message.deleted &&
				(this.message.saved || this.message.distributed || this.message.seen)
			)
		}
	},

	methods: {
		onHoverMessage() {
			this.imageHover = true
			this.messageHover = true
			if (this.canEditMessage()) this.hoverMessageId = this.message._id
		},
		canEditMessage() {
			return !this.message.deleted
		},
		onLeaveMessage() {
			this.imageHover = false
			if (!this.optionsOpened && !this.emojiOpened) this.messageHover = false
			this.hoverMessageId = null
		},
		openFile(action) {
			this.$emit('open-file', { message: this.message, action })
		},
		openUserTag(user) {
			this.$emit('open-user-tag', { user })
		},
		messageActionHandler(action) {
			this.messageHover = false
			this.hoverMessageId = null

			setTimeout(() => {
				this.$emit('message-action-handler', { action, message: this.message })
			}, 300)
		},
		checkVideoType(file) {
			if (!file) return
			const videoTypes = ['video/mp4', 'video/ogg', 'video/webm']
			const { type } = file
			return videoTypes.some(t => type.toLowerCase().includes(t))
		},
		sendMessageReaction({ emoji, reaction }) {
			this.$emit('send-message-reaction', {
				messageId: this.message._id,
				reaction: emoji,
				remove: reaction && reaction.indexOf(this.currentUserId) !== -1
			})
			this.messageHover = false
		}
	}
}
</script>

<style lang="scss" scoped>
.vac-card-info {
	border-radius: 4px;
	text-align: center;
	margin: 10px auto;
	font-size: 12px;
	padding: 4px;
	display: block;
	overflow-wrap: break-word;
	position: relative;
	white-space: normal;
	box-shadow: 0 1px 1px -1px rgba(0, 0, 0, 0.1),
		0 1px 1px -1px rgba(0, 0, 0, 0.11), 0 1px 2px -1px rgba(0, 0, 0, 0.11);
}

.vac-card-date {
	max-width: 150px;
	font-weight: 500;
	text-transform: uppercase;
	color: var(--chat-message-color-date);
	background: var(--chat-message-bg-color-date);
}

.vac-card-system {
	max-width: 250px;
	padding: 8px 4px;
	color: var(--chat-message-color-system);
	background: var(--chat-message-bg-color-system);
}

.vac-line-new {
	color: var(--chat-message-color-new-messages);
	position: relative;
	text-align: center;
	font-size: 13px;
	padding: 10px 0;
}

.vac-line-new:after,
.vac-line-new:before {
	border-top: 1px solid var(--chat-message-color-new-messages);
	content: '';
	left: 0;
	position: absolute;
	top: 50%;
	width: calc(50% - 60px);
}

.vac-line-new:before {
	left: auto;
	right: 0;
}

.vac-message-box {
	display: flex;
	flex: 0 0 50%;
	max-width: 50%;
	justify-content: flex-start;
	line-height: 1.4;
}

.vac-message-container {
	position: relative;
	padding: 2px 10px;
	align-items: end;
	min-width: 100px;
	box-sizing: content-box;
}

.vac-message-container-offset {
	margin-top: 10px;
}

.vac-offset-current {
	margin-left: 50%;
	justify-content: flex-end;
}

.vac-message-card {
	background: var(--chat-message-bg-color);
	color: var(--chat-message-color);
	border-radius: 8px;
	font-size: 14px;
	padding: 6px 9px 3px;
	white-space: pre-line;
	max-width: 100%;
	-webkit-transition-property: box-shadow, opacity;
	transition-property: box-shadow, opacity;
	transition: box-shadow 280ms cubic-bezier(0.4, 0, 0.2, 1);
	will-change: box-shadow;
	box-shadow: 0 1px 1px -1px rgba(0, 0, 0, 0.1),
		0 1px 1px -1px rgba(0, 0, 0, 0.11), 0 1px 2px -1px rgba(0, 0, 0, 0.11);
}

.vac-message-highlight {
	box-shadow: 0 1px 2px -1px rgba(0, 0, 0, 0.1),
		0 1px 2px -1px rgba(0, 0, 0, 0.11), 0 1px 5px -1px rgba(0, 0, 0, 0.11);
}

.vac-message-current {
	background: var(--chat-message-bg-color-me) !important;
}

.vac-message-deleted {
	color: var(--chat-message-color-deleted) !important;
	font-size: 13px !important;
	font-style: italic !important;
	background: var(--chat-message-bg-color-deleted) !important;
}

.vac-icon-deleted {
	height: 14px;
	width: 14px;
	vertical-align: middle;
	margin: -2px 2px 0 0;
	fill: var(--chat-message-color-deleted);
}

.vac-video-container {
	width: 350px;
	max-width: 100%;
	margin: 4px auto 5px;

	video {
		border-radius: 4px;
	}
}

::v-deep .vac-message-image {
	position: relative;
	background-color: var(--chat-message-bg-color-image) !important;
	background-size: cover !important;
	background-position: center center !important;
	background-repeat: no-repeat !important;
	height: 250px;
	width: 250px;
	max-width: 100%;
	border-radius: 4px;
	margin: 4px auto 5px;
	transition: 0.4s filter linear;
}

.vac-text-username {
	font-size: 13px;
	color: var(--chat-message-color-username);
	margin-bottom: 2px;
}

.vac-username-reply {
	margin-bottom: 5px;
}

.vac-text-timestamp {
	font-size: 10px;
	color: var(--chat-message-color-timestamp);
	text-align: right;
}

.selector:not(*:root),
#vac-audio-player {
	width: 250px;
	overflow: hidden;
	border-top-right-radius: 1em;
	border-bottom-right-radius: 2.5em 1em;

	audio {
		height: 40px;

		&::-webkit-media-controls-panel {
			height: 40px;
		}

		&::-webkit-media-controls-mute-button {
			display: none;
		}

		&::-webkit-media-controls-timeline {
			min-width: 103px;
			max-width: 142px;
		}

		&:focus {
			outline: none;
		}
	}
}

.vac-audio-message {
	margin-top: 3px;
}

.vac-file-message {
	display: flex;
	flex-wrap: wrap;
	align-items: center;
	margin-top: 3px;

	span {
		max-width: 100%;
	}

	.vac-icon-file svg {
		margin-right: 5px;
	}
}

.vac-icon-edited {
	-webkit-box-align: center;
	align-items: center;
	display: -webkit-inline-box;
	display: inline-flex;
	justify-content: center;
	letter-spacing: normal;
	line-height: 1;
	text-indent: 0;
	vertical-align: middle;
	margin: 0 4px 2px;

	svg {
		height: 12px;
		width: 12px;
	}
}

.vac-icon-check {
	height: 14px;
	width: 14px;
	vertical-align: middle;
	margin: -3px -3px 0 3px;
}

@media only screen and (max-width: 768px) {
	.vac-message-container {
		padding: 2px 3px 1px;
	}

	.vac-message-container-offset {
		margin-top: 10px;
	}

	.vac-message-box {
		flex: 0 0 80%;
		max-width: 80%;
	}

	.vac-offset-current {
		margin-left: 20%;
	}
}
</style>
