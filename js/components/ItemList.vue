<template>
	<div class="container wrapper">
		<div class="toolbar-wrapper">
			<div class="btn-toolbar pull-right" role="toolbar">
				<button id="sofl-clear" type="button" class="btn btn-default" @click="clearButtonClicked">
					<span class="glyphicon glyphicon-trash"></span>
				</button>
				<button id="sofl-export" type="button" class="btn btn-default" @click="exportButtonClicked">
					<span class="glyphicon glyphicon-export"></span>
				</button>
	  			<a class="btn btn-default" target="_blank" href="https://github.com/xralphack/Stackoverflow-Likes">
	    			<img class="github-icon" src="/img/github.png">
	  			</a>
			</div>
			<div class="btn-toolbar" role="toolbar">
				<button type="button" class="btn" :class="getTabClass(0)" @click="currentTab = 0;">
					<span class="glyphicon glyphicon-heart"></span>
				</button>
				<button type="button" class="btn" :class="getTabClass(1)" @click="currentTab = 1;">
					<span class="glyphicon glyphicon-bookmark"></span>
				</button>
			</div>
		</div>

		<div class="sofl-questions" v-show="currentTab == 0">
			<div class="row solf-content">
				<div class="col-md-4 col-sm-6" :key="item" v-for="(item, index) in items">
					<div class="sofl-card">
						<span class='glyphicon glyphicon-remove-circle delete-item-button' @click="deleteItem(index)"></span>
						<div class="sofl-card-header">
							{{ item.questionHeader }}
						</div>
						<hr>
						<div class="sofl-card-body" v-if="item.questionBody">
							{{ cleanItemBody(item.questionBody).substring(0, 250) }} ...
							<button class='btn btn-default sofl-card-read-button' @click="showItemDetail(item)">read more</button>
						</div>
					</div>
				</div>
			</div>

			<div class="row text-center solf-load" v-show="hasMore">
				<div class="col-md-4 col-md-offset-4">
					<button class="btn btn-primary btn-block solf-load-button" @click="displayItems(offset, limit)">Load More</button>
				</div>
			</div>
		</div>

		<div class="sofl-bookmarks" v-show="currentTab == 1">
			<div class="row">
				<div class="col-md-8">
					<div class="list-group">
					  <a target="_blank" :href="bookmark.url" type="button" class="list-group-item clearfix" v-for="(bookmark, index) in bookmarks">
					  	{{ bookmark.title }}
						<button type="button" class="btn btn-default pull-right" @click.prevent.stop="deleteBookmark(index)">
							<span class="glyphicon glyphicon-trash"></span>
						</button>
					  </a>
					</div>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
	export default {
		data() {
			return {
				currentTab: 0,
				limit: 6,
				offset: 0,
				hasMore: false,
				items: [],
				bookmarks: []
			};
		},
		methods: {
			displayItems(offset, limit) {
				this.hasMore = false;
				getQuestionsInStorage(offset, limit, (questions, hasMore) => {
					if (hasMore) {
						this.offset = this.offset + this.limit;
			      	}
			      	this.hasMore = hasMore;

			      	this.items.push.apply(this.items, questions);

			      	this.getItemsDetail(this.items);
				});
			},
			displayBookmarks() {
				getBookmarksInStorage((bookmarks) => {
					this.bookmarks = bookmarks;
				});
			},
			getItemsDetail(items) {
				this.getItemsRemoteData(this.getIds(items, 'question'), 'question');
				this.getItemsRemoteData(this.getIds(items, 'answer'), 'answer');
			},
			getItemsRemoteData(ids, type) {
				if (type == 'question') {
					getQuestionsDetail(ids, (data) => {
						this.showItemsRemoteData(data.items, type);
					});
				} else {
					getAnswersDetail(ids, (data) => {
						this.showItemsRemoteData(data.items, type);
					});
				}
			},
			showItemsRemoteData(items, type) {
				if (!items) { return; }
				var remoteQuestions = items;
	      		remoteQuestions.forEach((remoteQuestion) => {	
	      			var syncQuestions = this.items.filter((question) => {
	      				if (type == 'question') {
	      					return question.questionType == type && question.questionId == remoteQuestion.question_id;
	      				} else {
	      					return question.questionType == type && question.questionId == remoteQuestion.answer_id;
	      				}
					});

					var syncQuestion = syncQuestions[0];

					if (syncQuestion) {
						syncQuestion.questionBody = remoteQuestion.body;
						this.$set(this.items, this.items.indexOf(syncQuestion), syncQuestion);
					}
				});
			},
			getIds(items, type) {
				return items.filter((item) => {
					return item.questionType == type;
		      	}).map((item) => {
		      		return item.questionId;
		      	});
			},
			showItemDetail(item) {
				$('#myModalLabel').text(item.questionHeader);
				$('#bounceOut').attr("href", "http://stackoverflow.com/questions/" + item.questionId);
				$('.modal-body').html(item.questionBody).find('img').addClass('img-responsive');
				$('#myModal').modal('toggle');
			},
			deleteItem(index) {
				deleteQuestionById(this.items[index].questionId);
				this.items.splice(index, 1);
			},
			deleteBookmark(index) {
				deleteBookmark(this.bookmarks[index]);
				this.bookmarks.splice(index, 1);
			},
			cleanItemBody(body) {

				return $(body).text();
			},
			exportButtonClicked() {
				if (this.currentTab == 0) {
					var hash = 'likes';
				} else {
					var hash = 'bookmarks';
				}
				chrome.tabs.create({ url: chrome.runtime.getURL("src/override/export.html#" + hash) });
			},
			clearButtonClicked() {
				if (this.currentTab == 0) {
					this.hasMore = false;
					this.items = [];
					deleteAllQuestions();
				} else {
					this.bookmarks = [];
					deleteAllBookmarks();
				}
			},
			getTabClass(index) {
				return this.currentTab == index ? "btn-primary" : "btn-default";
			}
		},
		mounted() {
			this.displayItems(this.offset, this.limit);
			this.displayBookmarks();
		}
	}
</script>

<style lang="scss" scoped>
	.sofl-questions, .sofl-bookmarks {
		margin-bottom: 40px;
	}
	.toolbar-wrapper {
		margin-top: 20px;
		margin-bottom: 20px;
	}
	.github-icon {
		width: 20px;
  		height: 20px;
	}
	.sofl-card {
		background-color: #fff;
		height: 324px;
		border: 1px solid #d5d5d5;
		border-radius: 4px;
		margin-bottom: 20px;
		padding: 20px;
	}
	.sofl-card-header {
	    font-size: 20px;
	    color: #7f8c8d;
	}
	.sofl-card {
		position: relative;
	}
	.sofl-card-read-button {
		position: absolute;
		bottom: 20px;
		right: 20px;
	}
	.sofl-card .delete-item-button {
		opacity: 0;
		visibility: hidden;
		z-index: 10;
		position: absolute;
		top: -5px;
		right: -8px;
		color: #888;
		font-size: 24px;
		cursor: pointer;
		transition: opacity 0.5s;
	  	transition: visibility 0s, opacity 0.5s linear;
	}
	.sofl-card:hover .delete-item-button {
	  	visibility: visible;
	  	opacity: 1;
	}
</style>