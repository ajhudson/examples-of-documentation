(function ($) {	

	$.fn.tagManagerAdmin = function(userConfig) {

		this.render = function(jsonData) {
			render.call(this, jsonData);
		}

		var config = $.extend({
			companyId: null,
			compUserId: null,
			onLoadData: null,
			onLoadDataComplete: null
		}, userConfig);

		var NULL_HIERARCHY = "__nullhierarchy__";
		var maxRecurseLevel = 0;
		var isAppFormTag = false;
		var _this;
		var DIALOG_TYPE = {
			ADDOPTION_TAG: 1,
			ADDOPTION_CHILD: 2,
			DELETE_OPTIONS: 3,
			DELETE_TAG: 4,
			CONFIGURE_TAG: 5,
			ADD_TAG_CATEGORY: 6,
			CONFIGURE_APPFORM_TAG: 7
		};

		var ADD_ACTION = {
			ADD_TAG: "tag",
			ADD_OPTION: "option"
		};


		function setMaxRecurseLevel(newMaxRecurseLevel) {
			maxRecurseLevel = newMaxRecurseLevel
		}

		function getMaxRecurseLevel() {
			return maxRecurseLevel;
		}


		function setIsAppFormTag(isImported) {
			isAppFormTag = isImported;
		}


		function getIsAppFormTag() {
			return isAppFormTag;
		}


		function getMaxFromArray(arr) {
			return arr !== undefined ? Math.max.apply(Math, arr) : 0;
		}


		function annotateNoOptionsToWrapper(el) {
			return el.addClass("nooptions").css("height", getOptionControlCss().outerHeight + "px");
		}


		function sumArray(a) {
			var len = a.length,
				i = 0,
				sum = 0;

			for (i = 0; i < len; i++) {
				sum += a[i];
			}

			return sum;
		}


		function getOptionControlCss() {
			if (getOptionControlCss.css === undefined) {
				var el = $(".tag:first .optionControl:first"),
					h = parseInt(el.outerHeight()),
					m = parseInt(el.css("margin-bottom"));

				getOptionControlCss.css = {
					"outerHeight": (h + m),
					"margin": m,
					"height": h,
					"heightAndmargin": (h + m) + m
				};
			}

			return getOptionControlCss.css;
		}

		// create the HTML for a tag container div
		function createAdminTagHtml(tagData, iterationId, parentId) {
			var addTip = iterationId == 0 ? "Add new option to '" + tagData.NAME + "' tag" : iterationId == (getMaxRecurseLevel() - 1) ? "Add a new tag category underneath this level" : "";
			var addClass = iterationId == 0 ? "addTagOption" : iterationId == (getMaxRecurseLevel() - 1) ? "addNewTagCategory" : "";
			var canAdd = parseInt(tagData.ALLOWUSERSTOADDOPTIONS) == 1 ? true : false;
			var isMultipleChoice = parseInt(tagData.ISMULTIPLECHOICE) == 1 ? true : false;
			var showInApplicantManager = parseInt(tagData.SHOWINAPPLICANTMANAGER) == 1 ? true : false;
			var showInShortlistManager = parseInt(tagData.SHOWINSHORTLISTMANAGER) == 1 ? true : false;
			var deactivateTagHtml = iterationId > 0 ? "				<div class=\"wcTableCell tagHeader\"><a href=\"#\" class=\"deleteTagCategory\" data-tagid=\"" + tagData.TAGID + "\" title=\"Delete '" + tagData.NAME + "' tag and its children\"><span class=\"fa fa-trash\"></span></a></div>" : "";
			var addTagOptionHtml = "";
			var addChildTagHtml = "";
			var configureTagHtml = "";

			if (!getIsAppFormTag()) {
				configureTagHtml += "<div class=\"wcTableCell tagHeader\">";
				configureTagHtml += "<a href=\"#\" class=\"configureTagCategory\" data-tagid=\"" + tagData.TAGID + "\" title=\"Configure '" + tagData.NAME + "' tag\"><span class=\"fa fa-cog\"></span></a>";
				configureTagHtml += "</div>";					
				
				if (getMaxRecurseLevel() == 1) {
					configureTagHtml += "<div class=\"wcTableCell tagHeader\">";
					configureTagHtml += "<a href=\"#\" class=\"configureInitialTagHierarchy\" data-tagid=\"" + tagData.TAGID + "\" title=\"Configure '" + tagData.NAME + "' tag\"><span class=\"fa fa-plus\"></span></a>";
					configureTagHtml += "</div>";
				}
			}
			else {
				configureTagHtml += "<div class=\"wcTableCell tagHeader\">";
				configureTagHtml += "<a href=\"#\" class=\"configureAppFormTagCategory\" data-tagid=\"" + tagData.TAGID + "\" title=\"Configure '" + tagData.NAME + "' tag\"><span class=\"fa fa-cog\"></span></a>";
				configureTagHtml += "</div>";	
			}

			if (addTip.length > 0 && getIsAppFormTag() == false) {
				var addTagIcon = iterationId == (getMaxRecurseLevel() - 1) ? "<span class=\"fa fa-chevron-right\"></span>" : "<span class=\"fa fa-plus\"></span>";
				addTagOptionHtml = "				<div class=\"wcTableCell tagHeader\"><a href=\"#\" data-tagid=\"" + tagData.TAGID + "\" class=\"" + addClass + "\" title=\"" + addTip + "\">" + addTagIcon + "</a></div>";
			}

			var tagHtml = [
				"<div id=\"" + tagData.NAME.replace(" ", "") + "Tag\" class=\"tag\" data-iterationid=\"" + iterationId + "\" data-tagid=\"" + tagData.TAGID + "\" data-parentid=\"" + parentId + "\" data-canadd=\"" + canAdd + "\" data-ismultiplechoice=\"" + isMultipleChoice + "\" data-showinappmanager=\"" + showInApplicantManager + "\" data-showinshortlistmanager=\"" + showInShortlistManager + "\" data-tagname=\"" + tagData.NAME + "\">",
				"		<div class=\"wcTable tagTable\">",
				"			<div class=\"wcTableRow\">",
				"				<div class=\"wcTableCell tagHeader\">" + tagData.NAME + "</div>",
				configureTagHtml,
				addTagOptionHtml,
				addChildTagHtml,
				deactivateTagHtml,
				"			</div>",
				"		</div>",
				"		<div class=\"optionsCaption\">Options</div>",
				"</div>"
			].join("");

			return tagHtml;
		}

		// generate the HTML for an option div
		function createOptionsAdminHtml(option, iterationId, rootTagId, optionsWithoutChildrenCount, isLast) {
			var hasChildOptions = getObjectKeysLength(option.OPTIONS) > 0;
			var lastTagIndex = getMaxRecurseLevel() - 1;
			var hasChildOptionsAttr = "data-haschildren=\"" + hasChildOptions + "\" ";
			var addOptionHtml = iterationId < lastTagIndex ? "<span class=\"optionIcons addOption\"><a href=\"#\" title=\"Add option to '" + option.NAME + "'\" class=\"btnAddOption\"><span class=\"fa fa-plus\"></span></a></span>" : "";
			var deleteOptionHtml = getIsAppFormTag() == false ? "	<span class=\"optionIcons deleteOption\"><a href=\"#\" class=\"btnDeleteOption\" title=\"Delete '" + option.NAME + "' and its children\"><span class=\"fa fa-trash\"></span></a></span>" : "";
			var addedByUserClass = option.ADDEDVIATAGADMINMANAGER == false || option.ADDEDBYCLIENT == false ? "addedByUser" : "";
			
			var titleAttr = " title=\"";
			if (option.ISAPPFORMTAG) {
				titleAttr += "Imported from an application form question";
			} else if (option.ADDEDVIATAGADMINMANAGER) {
				titleAttr += "Added by a user in Tag Administrator";
			} else if (option.ADDEDBYCLIENT) {
				titleAttr += "Added by a user in Talent Pond Manager";
			} else {
				titleAttr += "This is a system tag";
			}
			titleAttr += "\"";

			var html = [
						"<div id=\"optionControlFor_" + option.OPTIONID + "\" ",
						"class=\"optionControl " + addedByUserClass  + "\" ",
						"data-optionid=\"" + option.OPTIONID + "\" ",
						"data-optiontext=\"" + option.NAME + "\" ",
						"data-parentid=\"" + option.PARENTID + "\" ",
						"data-rootoptionid=\"" + option.ROOTOPTIONID + "\" ",
						"data-tagid=\"" + option.TAGID + "\" ",
						hasChildOptionsAttr,
						"data-roottagid=\"" + rootTagId + "\" ",
						"data-iterationid=\"" + iterationId + "\" ",
						"data-islast=\"" + isLast + "\" ",
						titleAttr,
						"data-addedviaadmin=\"" + option.ADDEDVIATAGADMINMANAGER + "\">",
						"	<span class=\"optionName\">" + option.NAME + "</span>",
						deleteOptionHtml,
						addOptionHtml,
						"</div>"
					].join("");

			return html;
		}

		// find the tag container which is the tallest, and ensure all other tag containers follow suit
		function normaliseHeightForTags() {
			var tagEls = $(".tag");
			var heights = tagEls.map(function () {
				return $(this).height();
			});

			tagEls.css("height", parseInt(getMaxFromArray(heights)));
		}

		// sort DIV's based on attribute
		function sortByAttrFactory(attributeName) {
			var fn = function (elA, elB) {
				var a = $(elA).attr(attributeName);
				var b = $(elB).attr(attributeName);
				
				if (a > b) {
					return 1;
				}
				else if (a < b) {
					return -1;
				}
				else {
					return 0;
				}
			};

			return fn;
		}

		// create a div to wrap around options
		function createOptionWrapperDiv(parentId, optionText, rootOptionId, additional) {
			// don't create the div if it has already been created
			if ($("#optionsFor_" + parentId).length > 0) {
				return null;
			}

			var divAttribs = {
						"class": "optionsForWrapper",
						"id": "optionsFor_" + parentId,
						"data-parentid": parentId,
						"data-parentoptiontext": optionText,
						"data-rootoptionid": rootOptionId
				};

			if (additional !== undefined) {
				$.extend(divAttribs, additional);
			}

			return $("<div />", divAttribs);
		}

		// take an object, extract the keys, and sort the keys alphabetically
		function getSortedObjectKeys(obj) {
			var keys = [],
				k = "";

			for (k in obj) {
				keys.push(k);
			}

			return keys.sort();
		}

		// add the tag catgory containers to the page
		function renderTagCategoryContainers(el, tagCategories) {
			var i = 0,
				tag = {},
				adminTagHtml = "",
				tagParentId = 0;

			for (i = 0; i < tagCategories.length; i++) {
				var tag = tagCategories[i];
				var adminTagHtml = createAdminTagHtml(tag, i, tagParentId);
				el.append(adminTagHtml); // add the tag HTML to the page
				tagParentId = tag.TAGID;
			}
		}


		// add spacer divs in the right places so that the options are lined up correctly
		function renderHierarchyTagPositions(spacerDivsCache) {
			var s = 0,
				j = 0,
				currSpacerCache = {},
				spacerDiv = {},
				spacerDivAttr = {},
				el = {};

			for (s = 0; s < spacerDivsCache.length; s++) {
				currSpacerCache = spacerDivsCache[s];

				for (j = 0; j < currSpacerCache.count; j++) {
					spacerDivAttr = {
						"height": getOptionControlCss().outerHeight + "px", 
						"class": "spacerDiv", 
						id: "spacerDivFor_" + currSpacerCache.optionid + "_" + j
					};

					spacerDiv = $("<div />", spacerDivAttr);
					el = $("#" + currSpacerCache.elementid);
					el.after(spacerDiv);	
				}
			}
		}

		// add spacers adjacent to the last tag for which the tag hierarchy does not reach maximium depth
		function addSpacerDivsForShortHierarchy(currentHierarchyLevel, parentOptionText, optionId, rootOptionId) {
			var nextToLastLevel = getMaxRecurseLevel() - 1,
				nextRecurseLevel = currentHierarchyLevel + 1,
				i = 0,
				spacerAttr = {},
				blankWrapperDiv = {};

			if (currentHierarchyLevel < nextToLastLevel) {
				for (i = nextRecurseLevel; i < getMaxRecurseLevel(); i++) {
					var spacerAttr = {
						"class": "optionsForWrapper",
						"id": "optionsFor_" + optionId+ "_" + i,
						"data-parentid": optionId,
						"data-parentoptiontext": parentOptionText,
						"data-rootoptionid": rootOptionId,
						"data-isspacer": true
					};

					var blankWrapperDiv = $("<div />", spacerAttr);
					blankWrapperDiv = annotateNoOptionsToWrapper(blankWrapperDiv);
					$(".tag:eq(" + i + ")").append(blankWrapperDiv);
				}
			}
		}

		// add a spacer as a substitute for an option if there is a 'gap' in the hierarchy
		function addNullHierarchySpacer(optionsWithoutChildrenCount, tagId, parentId, rootOptionId) {
			var i = 0,
				nullHierarchyAttr = {},
				nullHierarchyEl = {};

			for (i = 0; i < optionsWithoutChildrenCount; i++) {
				var divId = "nullHierarchyFor_" + tagId + "_" + parentId + "_" + i;
				var cssClass = i == 0 ? "nullHierarchy parent" : "nullHierarchy";
				var nullHierarchyAttr = {
					"height": getOptionControlCss().outerHeight + "px", 
					"class": cssClass, 
					"data-parentid": parentId, 
					"data-rootoptionid": rootOptionId,
					"data-isnullhierarchyspacer": true,
					"id": divId
				};
				
				var nullHierarchyEl = $("<div />", nullHierarchyAttr);
				var tagEl = $(".tag[data-tagid='" + tagId + "']");
				tagEl.append(nullHierarchyEl);
			}
		}

		// after each tag category has been generated this function will add some extra divs to seperate out the tag categories clearly
		function seperateTagCategory(tagId) {
			var i = 0,
				tagIdSuffix = "",
				divAttr = {};

			for (i = 0; i < 2; i++) {
				$(".tag").each(function (index, el) {
					classSuffix = i == 1 ? "top" : "bottom";
					divAttr = {
						"id": "categorySeperatorFor_" + tagId + "_" + classSuffix + "_" + index,
						"class": "categorySeperator " + classSuffix
					};

					var seperatorDiv = $("<div />", divAttr);
					$(this).append(seperatorDiv);
				});
			}
		}


		// get the number of options in a tag hierarchy which do not have children. this helps us work out how many spacer divs we need to add to ensure the hierarchy is displayed correctly
		function getChildlessOptions(options, currentHierarchyLevel) {
			var count = 0,
				nextLevel = currentHierarchyLevel + 1,
				childOptsLen = getObjectKeysLength(options.OPTIONS);

			if (childOptsLen > 0) {
				traverseOptions(options.OPTIONS, nextLevel);
			}

			return count;

			// nested function which is called recursively to assist with generating the hierarchy
			function traverseOptions(traversedOpts, currentHierarchyLevel) {
				var o = {},
					nextLevel = currentHierarchyLevel + 1,
					thisChildOptsLen = 0;

				for (o in traversedOpts) {
					thisChildOptsLen = getObjectKeysLength(traversedOpts[o].OPTIONS);

					if (thisChildOptsLen) {
						traverseOptions(traversedOpts[o].OPTIONS, nextLevel);
					}
					else {
						count++;
					}
				}
			}
		}


		function getAllRootOptionIds() {
			var rootOptionIds = $(".tag:first .optionControl[data-rootoptionid]").map(function() {
				return parseInt($(this).attr("data-rootoptionid"));
			});
			return rootOptionIds;
		}


		// generate the HTML for a dialog window to add an option to an existing tag category
		function generateHtmlForAddChildOptionDialog(optionId) {
			var optionEl = $(".optionControl[data-optionid='" + optionId + "']");
			var rootOptionId = parseInt(optionEl.attr("data-rootoptionid"));
			var actionedFromTagEl = optionEl.closest(".tag");
			var actionedFromTagId = parseInt(actionedFromTagEl.attr("data-tagid"));
			var tagToAddOptionToEl = $(".tag[data-parentid='" + actionedFromTagId + "']");
			var tagId = parseInt(tagToAddOptionToEl.attr("data-tagid"));
			var optionText = optionEl.attr("data-optiontext");
			var currentLevel = parseInt(actionedFromTagEl.attr("data-iterationid"));
			var nextLevel = currentLevel + 1;
			var maxLevel = getMaxRecurseLevel() - 1;
			var existingOptionsInHierarchy = $(".tag:gt(" + currentLevel + ") .optionControl[data-rootoptionid='" + rootOptionId + "'][data-parentid='" + optionId + "']").length > 0;
			var askAttachToTag = ((maxLevel - currentLevel) >= 2 && !existingOptionsInHierarchy); // only ask which tag to attach the option to if there is more than one child tag categories underneath the current position
			var tags = [];
			var tagSelectionHtml = "";
			var nextTagEl = $(".tag[data-iterationid='" + nextLevel + "']");
			var currId = nextTagEl.length > 0 ? parseInt(nextTagEl.attr("data-tagid")) : 0;
			var nextTagElWithOptions = $(".tag:gt(" + currentLevel + ") .optionControl[data-rootoptionid='" + rootOptionId + "'][data-parentid='" + optionId + "']").closest(".tag");
			var nextTagIdWithOptions = nextTagElWithOptions.length > 0 ? parseInt(nextTagElWithOptions.attr("data-tagid")) : parseInt($(".tag:eq(" + nextLevel + ")").attr("data-tagid"));
			var currVal = "";

			if (askAttachToTag) {
				tags = $.makeArray($(".tag:gt(" + currentLevel + ")").map(function (i, e) {
					currId = parseInt($(this).attr("data-tagid"));
					currVal = $(this).attr("data-tagname");
					var checkedAttr = i == 0 ? " checked=\"checked\"" : "";
					return "<p><input type=\"radio\" name=\"attachToTag\" id=\"attachToTag_" + currId + "\" value=\"" + currId + "\"" + checkedAttr + "> " + currVal + "</p>";
				}));	
			}

			var tagSelectionHtml = tags.length > 1 ? 
										"<p><strong>Which tag do you want to associate the option with?</strong></p>" + tags.join("") : 
										"<input type=\"radio\" name=\"attachToTag\" id=\"attachToTag_" + nextTagIdWithOptions + "\" value=\"" + nextTagIdWithOptions + "\" checked=\"checked\" style=\"display: none;\"> ";

			var html = [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\" class=\"form-inline\">",
				"<p>Please give your new option under the <strong>" + optionText + "</strong> tag a name.</p>",
				"<p>The option must only contain alphanumeric characters but can contain spaces, and must be between 3 - 50 characters in length.</p>",
				"<p>The name must be unique and cannot be the same as another tag in this hierarchy.</p>",
				"<div class=\"wcTable\" id=\"addOptionTable\">",
				"</div>",
				"<div style=\"float: right;\">",
				"<input type=\"button\" class=\"btn btn-primary\" id=\"btnAddAnotherOption\" name=\"btnAddAnotherOption\" value=\"Add another\">&nbsp;",
				"<input type=\"button\" class=\"btn btn-primary\" id=\"btnRemoveLastOption\" name=\"btnRemoveLastOption\" value=\"Remove last option\" disabled=\"disabled\">",
				"</div>",
				"<div style=\"height: 10px;\"></div>",
				tagSelectionHtml,
				"<input type=\"hidden\" name=\"ParentOptionId\" id=\"ParentOptionId\" value=\"" + optionId + "\">",
				"</form>"
			];

			return html.join("");
		}


		function openLoadingDialog(titleCaption) {
			var newTitle = titleCaption !== undefined ? titleCaption : "Please wait until the operation has finished...";
			loadingDialog.setTitle(newTitle);
			loadingDialog.setBodyHtml(getLoadingDialogContent());

			actionDialog.hide();
			loadingDialog.show();
		}


		function closeLoadingDialog() {
			loadingDialog.hide();
		}

		function createModalDialog(dialogType, titleText, generatedHtml) {
			actionDialog.setTitle(titleText);
			actionDialog.setBodyHtml(generatedHtml);
			actionDialog.enableOkButton();

			var okCallback = null;
			var onShowCallback = null;

			switch (dialogType) {
				case DIALOG_TYPE.ADDOPTION_TAG:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var action = _modal.find("input[name='newTagOrNewOption']:checked").val();
						
						if (action !== undefined) {

							tagId = parseInt(_modal.find("#tagId").val());
							var tagEl = $(".tag[data-tagid='" + tagId + "']");

							switch (action.toLowerCase())
							{
								case ADD_ACTION.ADD_OPTION:
									askAddTagOptionDialog.call(tagEl, true);
									break;

								case ADD_ACTION.ADD_TAG:
									askAddNewTagCategoryDialog.call(tagEl);
									break;
							}
							return true;
						}

						var parentOptionId = 0;
						var newOptionName = _modal.find("#txtNewTagOptionName").val();
						var tagId = parseInt(_modal.find("input[name='attachToTag']").val());
						addNewOption(tagId, newOptionName, parentOptionId);
					};

					onShowCallback = function() {
						var _model = $("#actionDialog");
						_model.find("input[type='text']").first().focus();
					};

					break;

				case DIALOG_TYPE.ADDOPTION_CHILD:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var optionValues = $.makeArray(_modal.find("input[type='text'][id^='txtNewTagOptionName']").map(function() {
							return $(this).val();
						}));
						var tagId = parseInt(_modal.find("input[name='attachToTag'][type='radio']:checked").val());
						var parentOptionId = parseInt(_modal.find("#ParentOptionId").val());
						addMultipleNewOptions(tagId, optionValues, parentOptionId);
						return true;
					};

					onShowCallback = function() {
						var _modal = $("#actionDialog");
						_modal.find("input[type='text']").first().focus();
					};
					break;

				case DIALOG_TYPE.DELETE_OPTIONS:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var optionId = parseInt(_modal.find("#deleteOptionId").val());
						deleteOption(optionId);
						actionDialog.enableOkButton();
						return true;
					};
					break;

				case DIALOG_TYPE.DELETE_TAG:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var tagId = parseInt(_modal.find("#deleteTagId").val());
						deleteTagCategory(tagId);
						return true;
					};
					actionDialog.enableOkButton();
					break;

				case DIALOG_TYPE.CONFIGURE_TAG:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var isMultipleChoice = _modal.find("#isMultipleChoice:checked").length > 0 ? true : false;
						var usersCanAddOptions = _modal.find("#usersCanAddOptions:checked").length > 0 ? true : false;
						var tagId = parseInt(_modal.find("#configureTagId").val());
						updateTagConfig(tagId, usersCanAddOptions, isMultipleChoice);

						return true;
					};
					break;

				case DIALOG_TYPE.CONFIGURE_APPFORM_TAG:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var tagId = parseInt(_modal.find("#configureAppFormTagId").val());
						var showInAppManager = _modal.find("#showInApplicationManager:checked").length > 0 ? true : false;
						var showInShortlistManager = _modal.find("#showInShortlistManager:checked").length > 0 ? true : false;
						updateAppFormTagConfig(tagId, showInAppManager, showInShortlistManager);

						return true;
					}

					break;

				case DIALOG_TYPE.ADD_TAG_CATEGORY:
					okCallback = function() {
						if (!$("#frmAction").valid()) {
							return false;
						}

						var _modal = $("#actionDialog");
						var parentTagId = parseInt(_modal.find("#parentTagId").val());
						var tagName = _modal.find("#txtNewTagCategoryName").val();
						addNewTag(tagName, parentTagId);
						return true;
					};

					onShowCallback = function() {
						var _modal = $("#actionDialog");
						_modal.find("#txtNewTagCategoryName").focus();
					}
					break;
			}

			actionDialog.setOkCallback(okCallback);
			actionDialog.show(onShowCallback);
			
			// jquery form validation rules are determined here
			$("#frmAction").validate({
				rules: {
					txtNewTagOptionName: {
						required: true,
						rangelength: [3, 50],
						alphanumeric: true,
						uniquetagoroption: true
					},
					txtNewTagCategoryName: {
						required: true,
						rangelength: [3, 50],
						alphanumeric: true,
						uniquetagoroption: true
					}
				},
				errorElement: "span"
			});


			// add multiple options at the same time
			if (dialogType == DIALOG_TYPE.ADDOPTION_CHILD) {
				$("#btnAddAnotherOption").click(function() {
					addAnotherOptionTextBox();
				});

				$("#btnRemoveLastOption").click(function() {
					removeLastOptionTextBox();
				});

				addAnotherOptionTextBox();
			}
		}


		function addAnotherOptionTextBox() {
			var dialogContext = $("#actionDialog"),
				existingInputsCount = dialogContext.find(".addAnotherOptionRow").length,
				optionIndex = existingInputsCount,
				displayIndex = optionIndex + 1;

			var addOptionHtmlTemplate = [
				"	<div class=\"wcTableRow addAnotherOptionRow\" data-index=\"" + optionIndex + "\">",
				"		<div class=\"wcTableCell\" style=\"white-space: nowrap; padding-bottom: 10px; vertical-align: top \"><strong>New option name " + displayIndex + ":</strong></div>",
				"		<div class=\"wcTableCell\" style=\"width: 90%;\"><input type=\"text\" class=\"form-control\" name=\"txtNewTagOptionName_" + optionIndex + "\" id=\"txtNewTagOptionName_" + optionIndex + "\" data-index=\"" + optionIndex + "\"></div>",
				"	</div>"
			];

			var html = addOptionHtmlTemplate.join("");
			dialogContext.find("#addOptionTable").append(html);

			$("#txtNewTagOptionName_" + optionIndex).rules("add", {
				required: true,
				rangelength: [3, 50],
				alphanumeric: true,
				uniquetagoroption: true,
				uniquemultipleoption: true,
				messages: {
					required: "New option name " + displayIndex + " is required"
				}
			});

			// focus on the new text box
			$("#txtNewTagOptionName_" + optionIndex).focus();

			// enable the remove last option button if it is disabled
			if (existingInputsCount > 0) {
				$("#btnRemoveLastOption:disabled").removeProp("disabled");	
			}
		}

		function removeLastOptionTextBox() {
			var dialogContext = $("#actionDialog");
				lastOptionEl = $(".addAnotherOptionRow[data-index!='0']:last"),
				currentIndex = parseInt(lastOptionEl.attr("data-index")),
				textBoxEl = lastOptionEl.find("input[type='text']"),
				lastOptionExists = lastOptionEl.length > 0;

				if (lastOptionExists) {
					textBoxEl.rules("remove"); // remove the validation rules from the text box we are removing
					lastOptionEl.remove(); // remove the element from the dialog

					// if the data-index attrinute of the element we just removed was '1', that should mean the there is one element left with data-index attribute of '0' - disable the 'remove last option' button
					if (currentIndex == 1) {
						dialogContext.find("#btnRemoveLastOption").prop("disabled", "disabled");
					}
				}
		}

		function generateHtmlForAddTagOptionDialog(tagId, explicitlyAskForOptionName) {
			var tagName = $(".tag[data-tagid='" + tagId + "']").attr("data-tagname");

			var html = (getMaxRecurseLevel() > 1 || explicitlyAskForOptionName === true) ? [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\" class=\"form-inline\">",
				"<p>Please give your new option under the <strong>" + tagName + "</strong> tag a name.</p>",
				"<p>The option must only contain alphanumeric characters but can contain spaces, and must be between 3 - 50 characters in length.</p>",
				"<p>The name must be unique and cannot be the same as another tag or option in this hierarchy.</p>",
				"<p><strong>New option name:</strong> <input type=\"text\" class=\"form-control\" name=\"txtNewTagOptionName\" id=\"txtNewTagOptionName\" maxlength=\"50\" size=\"50\"></p>",
				"<input type=\"hidden\" name=\"attachToTag\" id=\"TagId\" value=\"" + tagId + "\">",
				"</form>"
			] : [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\" class=\"form-inline\">",
				"<p>Firstly, decide if you want to add a new option under <strong>'" + tagName + "'</strong>, or add a new child tag by choosing one of the options below:</p>",
				"<p><input type=\"radio\" name=\"newTagOrNewOption\" id=\"newTagOrNewOption_tag\" value=\"" + ADD_ACTION.ADD_TAG + "\">&nbsp;Create a new tag as a child to '" + tagName + "'</p>",
				"<p><input type=\"radio\" name=\"newTagOrNewOption\" id=\"newTagOrNewOption_option\" value=\"" + ADD_ACTION.ADD_OPTION + "\" checked=\"checked\">&nbsp;Create a new option under '" + tagName + "'</p>",
				"<input type=\"hidden\" name=\"tagId\" id=\"tagId\" value=\"" + tagId + "\">",
				"</form>"
			];

			return html.join("");
		}


		function generateHtmlForDeleteOptionDialog(optionId) {
			var optionText = $(".optionControl[data-optionid='" + optionId + "']").attr("data-optiontext");
			var html = [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\">",
				"<p>Are you sure you want to delete the option for <strong>'" + optionText + "'</strong> and all of its children?",
				"<p>Click <strong>OK</strong> to confirm or click <strong>Cancel</strong> keep the option.</p>",
				"<input type=\"hidden\" name=\"deleteOptionId\" id=\"deleteOptionId\" value=\"" + optionId + "\">",
				"</form>"
			];	

			return html.join("");
		}


		function generateHtmlForDeleteTagDialog(tagId) {
			var tagName = $(".tag[data-tagid='" + tagId + "']").attr("data-tagname");
			var html = [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\">",
				"<p>Are you sure you want to delete the entire tag category for <strong>'" + tagName + "'</strong> and all of its child tag categories and associated options?",
				"<p>Click <strong>OK</strong> to confirm or click <strong>Cancel</strong> keep the tag category.</p>",
				"<input type=\"hidden\" name=\"deleteTagId\" id=\"deleteTagId\" value=\"" + tagId + "\">",
				"</form>"
			];	

			return html.join("");
		}


		function  generateHtmlForConfigTagCategory(tagId) {
			var tagEl = $(".tag[data-tagid='" + tagId + "']"),
				tagName = tagEl.attr("data-tagname"),
				isMultipleChoice = tagEl.attr("data-ismultiplechoice") == "true",
				canAdd = tagEl.attr("data-canadd") == "true",
				isMultipleChoiceAttr = isMultipleChoice ? " checked=\"checked\"" : "";
				canAddAttr = canAdd ? " checked=\"checked\"" : "";
				
			var multipleChoiceHtml = "<p><input type=\"checkbox\" name=\"isMultipleChoice\" id=\"isMultipleChoice\" " + isMultipleChoiceAttr + " value=\"true\">&nbsp;Is multiple choice?</p>";
			
			var html = [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\">",
				"<p>You can configure the behaviour of options under the <strong>'" + tagName + "'</strong> tag here.",
				multipleChoiceHtml,
				"<p><input type=\"checkbox\" name=\"usersCanAddOptions\" id=\"usersCanAddOptions\" " + canAddAttr + " value=\"true\">&nbsp;Users can add options?</p>",
				"<p>Click <strong>OK</strong> to make changes or click <strong>Cancel</strong> to disregard.</p>",
				"<input type=\"hidden\" name=\"configureTagId\" id=\"configureTagId\" value=\"" + tagId + "\">",
				"</form>"
			];

			return html.join("");
		}

		function generateHtmlForConfigAppFormCategory(tagId) {
			var tagEl = $(".tag[data-tagid='" + tagId + "']"),
				showInAppManager = tagEl.attr("data-showinappmanager") == "true",
				showInShortlistManager = tagEl.attr("data-showinshortlistmanager") == "true",
				showInAppManagerAttr = showInAppManager ? " checked=\"checked\"" : "",
				showInShortlistManagerAttr = showInShortlistManager ? " checked=\"checked\"" : "",
				tagName = tagEl.attr("data-tagname");

			var html = [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\">",
				"<p><input type=\"checkbox\" name=\"showInApplicationManager\" id=\"showInApplicationManager\" " + showInAppManagerAttr + " value=\"true\">&nbsp;Show in application manager?</p>",
				"<p><input type=\"checkbox\" name=\"showInShortlistManager\" id=\"showInShortlistManager\" " + showInShortlistManagerAttr + " value=\"true\">&nbsp;Show in shortlist manager?</p>",
				"<p>Click <strong>OK</strong> to make changes or click <strong>Cancel</strong> to disregard.</p>",
				"<input type=\"hidden\" name=\"configureAppFormTagId\" id=\"configureAppFormTagId\" value=\"" + tagId + "\">",
				"</form>"
			];

			return html.join("");
		}

		function generateHtmlForAddNewTagCategory(tagId) {
			var tagName = $(".tag[data-tagid='" + tagId + "']").attr("data-tagname");
			var html = [
				"<form name=\"frmAction\" id=\"frmAction\" onSubmit=\"return false;\" class=\"form-inline\">",
				"<p>Please give your new tag category to sit under the <strong>" + tagName + "</strong> tag a name.</p>",
				"<p>The option must only contain alphanumeric characters but can contain spaces, and must be between 3 - 50 characters in length.</p>",
				"<p>The name must be unique and cannot be the same as another tag or option in this hierarchy.</p>",
				"<p><strong>New tag name:</strong> <input type=\"text\" class=\"form-control\" name=\"txtNewTagCategoryName\" id=\"txtNewTagCategoryName\" maxlength=\"50\"></p>",
				"<input type=\"hidden\" name=\"parentTagId\" id=\"parentTagId\" value=\"" + tagId + "\">",
				"</form>"
			];

			return html.join("");
		}


		function askAddTagOptionDialog(explicitlyAskForOptionName) {
			var tagId = parseInt($(this).attr("data-tagid"));
			var tagName = $(this).closest(".tag").attr("data-tagname");
			createModalDialog(DIALOG_TYPE.ADDOPTION_TAG, "Add option to the '" + tagName + "' tag", generateHtmlForAddTagOptionDialog(tagId, explicitlyAskForOptionName));
		}


		function askAddChildOptionDialog() {
			var optionId = parseInt($(this).parent().attr("data-optionid"));
			createModalDialog(DIALOG_TYPE.ADDOPTION_CHILD, "Add an option to an existing tag", generateHtmlForAddChildOptionDialog(optionId));
		}


		function askDeleteOption() {
			var optionId = $(this).parent().attr("data-optionid");
			var optionText = $(this).parent().attr("data-optiontext");
			createModalDialog(DIALOG_TYPE.DELETE_OPTIONS, "Delete option '" + optionText + "'?", generateHtmlForDeleteOptionDialog(optionId));
		}


		function askDeleteTagCategory() {
			var tagEl = $(this).closest(".tag");
			var tagId = parseInt(tagEl.attr("data-tagid"));
			var tagName = tagEl.attr("data-tagname");
			createModalDialog(DIALOG_TYPE.DELETE_TAG, "Delete tag category '" + tagName + "'?", generateHtmlForDeleteTagDialog(tagId));
		}


		function askConfigureTagCategoryDialog() {
			var tagEl = $(this).closest(".tag");
			var tagId = parseInt(tagEl.attr("data-tagid"));
			var tagName = tagEl.attr("data-tagname");
			createModalDialog(DIALOG_TYPE.CONFIGURE_TAG, "Configure tag category '" + tagName + "'", generateHtmlForConfigTagCategory(tagId));
		}

		function askConfigureAppFormTagCategory() {
			var tagEl = $(this).closest(".tag");
			var tagId = parseInt(tagEl.attr("data-tagid"));
			var tagName = tagEl.attr("data-tagname");
			createModalDialog(DIALOG_TYPE.CONFIGURE_APPFORM_TAG, "Configure imported question '" + tagName + "'", generateHtmlForConfigAppFormCategory(tagId));
		}

		function askAddNewTagCategoryDialog() {
			var tagEl = $(this).closest(".tag");
			var tagId = parseInt(tagEl.attr("data-tagid"));
			createModalDialog(DIALOG_TYPE.ADD_TAG_CATEGORY, "Add new tag category", generateHtmlForAddNewTagCategory(tagId));
			//$("#actionDialog").dialog("open");
		}


		function addNewOption(tagId, optionText, parentId) {
			openLoadingDialog("Please wait whilst a new option is created...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { refreshHierarchyCallback (response); });
			tm.setErrorHandler(genericErrorHandler);
			tm.AddNewOption(config.companyId, config.compUserId, tagId, optionText, parentId, true);
		}


		function addMultipleNewOptions(tagId, optionsToAdd, parentId) {
			openLoadingDialog("Please wait whilst the options are created...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { refreshHierarchyCallback (response); });
			tm.setErrorHandler(genericErrorHandler);
			tm.AddMultipleNewOptions(config.companyId, config.compUserId, tagId, optionsToAdd, parentId, true);
		}


		function addNewTag(tagName, parentId) {
			openLoadingDialog("Please wait whilst a new tag category is created...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { refreshHierarchyCallback (response); });
			tm.setErrorHandler(genericErrorHandler);
			tm.AddNewTag(config.companyId, config.compUserId, tagName, parentId, true);
		}


		function deleteOption(optionId) {
			openLoadingDialog("Please wait whilst the option is removed...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { refreshHierarchyCallback (response); });
			tm.setErrorHandler(genericErrorHandler);
			tm.DeleteOption(config.companyId, config.compUserId, optionId);
		}


		function deleteTagCategory(tagId) {
			openLoadingDialog("Please wait whilst the tag is removed...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { refreshHierarchyCallback (response); });
			tm.setErrorHandler(genericErrorHandler);
			tm.DeleteTagCategory(config.companyId, config.compUserId, tagId);
		}

		function updateTagConfig(tagId, usersCanAddOptions, isMultipleChoice) {
			openLoadingDialog("Please wait whilst tag configuration is updated...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { setTagConfigResponse(response, tagId, usersCanAddOptions, isMultipleChoice); });
			tm.setErrorHandler(genericErrorHandler);
			tm.SetTagSettings(config.companyId, config.compUserId, tagId, usersCanAddOptions, isMultipleChoice);
		}

		function updateAppFormTagConfig(tagId, showInAppManager, showInShortlistManager) {
			openLoadingDialog("Please wait whilst tag configuration is updated...");
			var tm = new TagManager();
			tm.setAsyncMode();
			tm.setCallbackHandler(function (response) { setAppFormTagSettingsResponse(response, tagId, showInAppManager, showInShortlistManager); });
			tm.setErrorHandler(genericErrorHandler);
			tm.SetAppFormTagSettings(config.companyId, config.compUserId, tagId, showInAppManager, showInShortlistManager);
		}

		function refreshHierarchyCallback(response) {
			closeLoadingDialog();

			if (!response.SUCCESS) {
				alert("Error! \r\n" + response.ERRMSG);
			}
			else {
				renderHierarchy(true);
			}
		}

		function setTagConfigResponse(response, tagId, canAdd, isMultipleChoice) {
			$(".tag[data-tagid='" + tagId + "']").attr("data-canadd", canAdd);
			$(".tag[data-tagid='" + tagId + "']").attr("data-ismultiplechoice", isMultipleChoice);
			closeLoadingDialog();
		}

		function setAppFormTagSettingsResponse(response, tagId, showInAppManager, showInShortlistManager) {
			$(".tag[data-tagid='" + tagId + "']").attr("data-showinappmanager", showInAppManager);
			$(".tag[data-tagid='" + tagId + "']").attr("data-showinshortlistmanager", showInShortlistManager);
			closeLoadingDialog();
		}

		function renderHierarchy(isRefresh) {

			// nested function to traverse options
			function traverseTagOptions(options, currentHierarchyLevel, rootTagId, parentWasNullHierarchy) {
				var o = "",
					i = 0,
					j = 0,
					keys = getSortedObjectKeys(options); // as JSON supplied from CF is not in the correct order, we need to re-order the keys alphabetically ourselves

				for (i = 0; i < keys.length; i++) {
					var tagName = keys[i];
					var o = options[tagName];
					var rootOptionId = o.ROOTOPTIONID;
					var optsWithoutChildrenCount = getChildlessOptions(o, currentHierarchyLevel);
					var spacersToAdd = optsWithoutChildrenCount - 1;

					if (currentHierarchyLevel == 0) {
						rootTagId = o.TAGID; // we want to connect all child options with the originating source option, we'll use this in a data attribute
					}

					var hasChildOptions = getObjectKeysLength(o.OPTIONS) > 0; // does the current option have any children?
					var isLast = i == (keys.length - 1);
					var isNullHierarchy = tagName == NULL_HIERARCHY;

					if (!isNullHierarchy) {
						var optionHtml = createOptionsAdminHtml(o, currentHierarchyLevel, rootTagId, optsWithoutChildrenCount, isLast); // create an option div
						var tagEl = $(".tag[data-tagid='" + o.TAGID + "']"); // find the tag container we are interested in
						tagEl.append(optionHtml); // add the option div to the page

						// if we are not at the last hierarchy level, we need to add spacer divs underneath so the options will line up properly. we'll do this at the end of the traversal process so for now store what we need in an array
						if (currentHierarchyLevel < getMaxRecurseLevel()) {
							spacersToAddCache.push({
								elementid: "optionControlFor_" + o.OPTIONID,
								optionid: o.OPTIONID,
								count: spacersToAdd
							});
						}
					}
					else {
						addNullHierarchySpacer(optsWithoutChildrenCount, o.TAGID, o.PARENTID, rootOptionId); // if this is a 'gap' in the hierarchy we need to add a spacer div now and then continue with the traversal
					}
					
					var nextRecurseLevel = currentHierarchyLevel + 1;
					if (hasChildOptions) {
						traverseTagOptions(o.OPTIONS, nextRecurseLevel, rootTagId, isNullHierarchy); // recurse here to get child options
					}
					else {
						addSpacerDivsForShortHierarchy(currentHierarchyLevel, tagName, o.OPTIONID, rootOptionId); // if there are no child options, but we've not reached full depth in the hierarchy, we need spacer divs
					}

					if (isLast && o.NAME.length > 0) {
						$(".tag").each(function (i, e) {
							var tagBarrierId = o.TAGID + "_" + i;
							$(this).append("<div class=\"tagBarrier\" id=\"" + tagBarrierId + "\"></div>");
						});
					}

					// add some more divs with a class of 'tagSeperator' to seperate each category with visible
					if (currentHierarchyLevel == 0 && i < (keys.length - 1)) {
						seperateTagCategory(o.TAGID); 
					}
				}
			}

			_this.html(""); // clear down the HTML

			if ($("#loadingDialog").length == 0) {
				_this.after($("<div />", { id: "loadingDialog" }));
			}

			if ($.isFunction(config.onLoadData)) {
				jsonData = JSON.parse(config.onLoadData.call(this, isRefresh));
			}

			if (jsonData == null) {
				alert("No data to generate hierarchy");
				return;
			}

			var currentRecurseLevel = 0,
				tagsInfo = jsonData.TAGS,
				rootTags = jsonData.DATA.OPTIONS,
				spacersToAddCache = [];

			setIsAppFormTag(jsonData.ISAPPFORM);

			setMaxRecurseLevel(tagsInfo.length); // set global variable
			renderTagCategoryContainers(_this, tagsInfo); // create tag category containers and add them to the page
			traverseTagOptions(jsonData.DATA.OPTIONS, currentRecurseLevel); // add the options to the page
			renderHierarchyTagPositions(spacersToAddCache);
			normaliseHeightForTags(); // make sure all tags have the same height

			// add some divs that we can use for dialog windows
			if ($("#actionDialog").length == 0) {
				_this.append($("<div />", { id: "actionDialog", "class": "modal fade", "tabindex": "-1", role: "dialog" }));
			}

			// add a new child option which belongs to a parent option
			$(".addOption").click(function (event) {
				askAddChildOptionDialog.call(this);
				event.preventDefault();
			});

			// add a new option to a tag
			$(".addTagOption").click(function (event) {
				if (getMaxRecurseLevel() == 1) {
					askAddNewTagCategoryDialog.call(this);
				} else {
					askAddTagOptionDialog.call(this);
				}
				event.preventDefault();
			});

			// deactivate any options
			$(".deleteOption").click(function (event) {
				askDeleteOption.call(this);
				event.preventDefault();
			});

			// delete a tag category and all of its options
			$(".deleteTagCategory").click(function(event) {
				askDeleteTagCategory.call(this);
				event.preventDefault();
			});

			// configure tag category
			$(".configureTagCategory").click(function(event) {
				askConfigureTagCategoryDialog.call(this);
				event.preventDefault();
			});

			// configure app form tag category
			$(".configureAppFormTagCategory").click(function(event) {
				askConfigureAppFormTagCategory.call(this);
				event.preventDefault();	
			});

			// configure initial tag hierarchy
			$(".configureInitialTagHierarchy").click(function(event) {
				askAddTagOptionDialog.call(this, true);
				event.preventDefault();
			});

			// add new tag category 
			$(".addNewTagCategory").click(function(event) {
				askAddNewTagCategoryDialog.call(this);
				event.preventDefault();
			});

			if ($.isFunction(config.onLoadDataComplete)) {
				config.onLoadDataComplete.call(this);
			}
		}

		return this.each(function () {
			_this = $(this);
			renderHierarchy(false);
		});
	};

}( jQuery ));