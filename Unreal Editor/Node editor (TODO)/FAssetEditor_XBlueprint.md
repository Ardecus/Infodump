```cpp
class FAssetEditor_XBlueprint : public FAssetEditorToolkit, public FNotifyHook, public FGCObject
{
public:
	virtual void InitXBlueprintAssetEditor(const EToolkitMode::Type Mode, const TSharedPtr<IToolkitHost>& InitToolkitHost, UXBlueprint* Blueprint);

	// IToolkit interface
	void RegisterTabSpawners(const TSharedRef<FTabManager>& InTabManager) override;
	void UnregisterTabSpawners(const TSharedRef<FTabManager>& InTabManager) override;
	// End of IToolkit interface

	// FAssetEditorToolkit
	FName GetToolkitFName() const override;
	FText GetBaseToolkitName() const override;
	FText GetToolkitName() const override;
	FText GetToolkitToolTipText() const override;
	FString GetWorldCentricTabPrefix() const override;
	FString GetDocumentationLink() const override;
	FLinearColor GetWorldCentricTabColorScale() const override;
	// End of FAssetEditorToolkit

	// FSerializableObject interface
	void AddReferencedObjects(FReferenceCollector& Collector) override;
	// End of FSerializableObject interface

	virtual FString GetReferencerName() const override
	{
		return TEXT("FAssetEditor_XBlueprint");
	}

	void SelectNode(const FString& NodeName);
	
	// not necessary, made for fancy magic
	static FString GetNodesText(const FGraphPanelSelectionSet& SelectedNodes);

protected:
	FName XBlueprintEditorAppName;
	FName XBlueprintPropertySID;
	FName SearchTabSID;
	FName EditorLayoutName;
	FName ViewportSID;
	FText WorkspaceMenuCategoryText;

	FName ToolkitName;
	FText BlueprintName;
	FString DocumentationLink;

	UXBlueprint* CurrentEditingXBlueprint = nullptr;
	TSharedPtr<FAssetEditorToolbar_XBlueprint> ToolbarBuilder;

	// Command list for this editor
	TSharedPtr<FUICommandList> GraphEditorCommands;

	TUniquePtr<FXSearch> SearchComponent;

	TSharedPtr<SGraphEditor> GetCurrentGraphEditor() const;
	FGraphPanelSelectionSet GetSelectedNodes() const;

	virtual void CreateCommandList();
	virtual void CreateGraph();
	virtual void CreateToolbar();
	virtual void BindCommands();
	virtual UClass* GetBlueprintClass() const PURE_VIRTUAL(, return UXBlueprint::StaticClass(););
	virtual UClass* GetGraphClass() const PURE_VIRTUAL(, return UXGraph::StaticClass(););
	virtual UClass* GetSchemaClass() const PURE_VIRTUAL(, return UAssetGraphSchema_XBlueprint::StaticClass(););
	virtual UClass* GetNodeClass() const PURE_VIRTUAL(, return UXNode::StaticClass(););

private:
	TSharedPtr<SGraphEditor> ViewportWidget;
	TSharedPtr<IDetailsView> PropertyWidget;
	bool bSearchTabVisible = false;

	TSharedRef<SDockTab> SpawnViewportTab(const FSpawnTabArgs& Args);
	TSharedRef<SDockTab> SpawnDetailsTab(const FSpawnTabArgs& Args);
	TSharedRef<SDockTab> SpawnSearchTab(const FSpawnTabArgs& Args);

	void CreateInternalWidgets();
	TSharedRef<SGraphEditor> CreateViewportWidget();

	// Delegates for graph editor commands
	void SelectAllNodes();
	bool CanSelectAllNodes() const;
	void DeleteSelectedNodes();
	bool CanDeleteNodes() const;
	void DeleteSelectedDuplicatableNodes();
	void CutSelectedNodes();
	bool CanCutNodes() const;
	void CopySelectedNodes();
	bool CanCopyNodes() const;
	void PasteNodes();
	void PasteNodesHere(const FVector2D& Location);
	bool CanPasteNodes() const;
	void DuplicateNodes();
	bool CanDuplicateNodes() const;
	void OnRenameNode();
	bool CanRenameNodes() const;
	void CreateCommentNode();
	bool CanCreateCommentNode() const;
	void ToggleSearchTab();
	bool CanToggleSearchTab() const;

	// Graph editor events
	void OnSelectedNodesChanged(const TSet<class UObject*>& NewSelection);
	void OnNodeTitleCommitted(const FText& NewText, ETextCommit::Type CommitInfo, UEdGraphNode* NodeBeingChanged);
	void OnFinishedChangingProperties(const FPropertyChangedEvent& PropertyChangedEvent);

	// not necessary, made for fancy magic
	TSet<UEdGraphNode*> ImportNodesFromText(UEdGraph* EdGraph, const FString& TextToImport);
};
```

```cpp
void FAssetEditor_XBlueprint::InitXBlueprintAssetEditor(const EToolkitMode::Type Mode, const TSharedPtr<IToolkitHost>& InitToolkitHost, UXBlueprint* Blueprint)
{
	CurrentEditingXBlueprint = Blueprint;
	CreateGraph();
	
	FGenericCommands::Register();
	FGraphEditorCommands::Register();
	FEditorCommands_XBlueprint::Register();

	CreateToolbar();

	BindCommands();
	CreateInternalWidgets();

	TSharedPtr<FExtender> ToolbarExtender = MakeShared<FExtender>();
	ToolbarBuilder->AddXBlueprintToolbar(ToolbarExtender);

	// Layout
	const TSharedRef<FTabManager::FLayout> StandaloneDefaultLayout = FTabManager::NewLayout(EditorLayoutName)
	->AddArea
	(
		FTabManager::NewPrimaryArea()->SetOrientation(Orient_Vertical)
		->Split
		(
			FTabManager::NewStack()
			->SetSizeCoefficient(0.1f)
			->AddTab(SearchTabSID, ETabState::OpenedTab)
		)
		->Split
		(
			FTabManager::NewSplitter()->SetOrientation(EOrientation::Orient_Horizontal)
			->Split
			(
				FTabManager::NewStack()
				->SetSizeCoefficient(0.8f)
				->AddTab(ViewportSID, ETabState::OpenedTab)
				->SetHideTabWell(true)
			)
			->Split
			(
				FTabManager::NewStack()
				->SetSizeCoefficient(0.2f)
				->AddTab(XBlueprintPropertySID, ETabState::OpenedTab)
				->SetHideTabWell(true)
			)
		)
	);

	const bool bCreateDefaultStandaloneMenu = true;
	const bool bCreateDefaultToolbar = true;
	FAssetEditorToolkit::InitAssetEditor(Mode, InitToolkitHost, XBlueprintEditorAppName, StandaloneDefaultLayout, bCreateDefaultStandaloneMenu, bCreateDefaultToolbar, CurrentEditingXBlueprint, false);

	RegenerateMenusAndToolbars();
}

void FAssetEditor_XBlueprint::RegisterTabSpawners(const TSharedRef<FTabManager>& InTabManager)
{
	WorkspaceMenuCategory = InTabManager->AddLocalWorkspaceMenuCategory(WorkspaceMenuCategoryText);
	TSharedRef<FWorkspaceItem> WorkspaceMenuCategoryRef = WorkspaceMenuCategory.ToSharedRef();
	FAssetEditorToolkit::RegisterTabSpawners(InTabManager);

	InTabManager->RegisterTabSpawner(ViewportSID, FOnSpawnTab::CreateSP(this, &FAssetEditor_XBlueprint::SpawnViewportTab))
	            .SetDisplayName(LOCTEXT("GraphCanvasTab", "Viewport"))
	            .SetGroup(WorkspaceMenuCategoryRef)
	            .SetIcon(FSlateIcon(FXEditorStyle::GetStyleSetName(), "GraphEditor.EventGraph_16x"));

	InTabManager->RegisterTabSpawner(XBlueprintPropertySID, FOnSpawnTab::CreateSP(this, &FAssetEditor_XBlueprint::SpawnDetailsTab))
	            .SetDisplayName(LOCTEXT("DetailsTab", "Property"))
	            .SetGroup(WorkspaceMenuCategoryRef)
	            .SetIcon(FSlateIcon(FXEditorStyle::GetStyleSetName(), "LevelEditor.Tabs.Details"));

	InTabManager->RegisterTabSpawner(SearchTabSID, FOnSpawnTab::CreateSP(this, &FAssetEditor_XBlueprint::SpawnSearchTab))
	            .SetDisplayName(LOCTEXT("SearchTab", "Search"))
	            .SetGroup(WorkspaceMenuCategoryRef)
	            .SetIcon(FSlateIcon(FXEditorStyle::GetStyleSetName(), "Kismet.Tabs.FindResults"));
}

void FAssetEditor_XBlueprint::UnregisterTabSpawners(const TSharedRef<FTabManager>& InTabManager)
{
	FAssetEditorToolkit::UnregisterTabSpawners(InTabManager);

	InTabManager->UnregisterTabSpawner(ViewportSID);
	InTabManager->UnregisterTabSpawner(XBlueprintPropertySID);
	InTabManager->UnregisterTabSpawner(SearchTabSID);
}

FName FAssetEditor_XBlueprint::GetToolkitFName() const
{
	return ToolkitName;
}

FText FAssetEditor_XBlueprint::GetBaseToolkitName() const
{
	return WorkspaceMenuCategoryText;
}

FText FAssetEditor_XBlueprint::GetToolkitName() const
{
	return FText::FromString(CurrentEditingXBlueprint->GetName());
}

FText FAssetEditor_XBlueprint::GetToolkitToolTipText() const
{
	return FAssetEditorToolkit::GetToolTipTextForObject(CurrentEditingXBlueprint);
}

FLinearColor FAssetEditor_XBlueprint::GetWorldCentricTabColorScale() const
{
	return FLinearColor::White;
}

FString FAssetEditor_XBlueprint::GetWorldCentricTabPrefix() const
{
	return ToolkitName.ToString();
}

FString FAssetEditor_XBlueprint::GetDocumentationLink() const
{
	return DocumentationLink;
}

void FAssetEditor_XBlueprint::AddReferencedObjects(FReferenceCollector& Collector)
{
	if (CurrentEditingXBlueprint)
	{
		Collector.AddReferencedObject(CurrentEditingXBlueprint);
		Collector.AddReferencedObject(CurrentEditingXBlueprint->Graph);
	}
}

void FAssetEditor_XBlueprint::CreateToolbar()
{
	if (!ToolbarBuilder.IsValid())
	{
		ToolbarBuilder = MakeShared<FAssetEditorToolbar_XBlueprint>(SharedThis(this));
	}
}

TSharedRef<SDockTab> FAssetEditor_XBlueprint::SpawnViewportTab(const FSpawnTabArgs& Args)
{
	ensure(Args.GetTabId() == ViewportSID);

	TSharedRef<SDockTab> SpawnedTab = SNew(SDockTab)
	.Label(LOCTEXT("ViewportTab_Title", "Viewport"));

	if (ViewportWidget.IsValid())
	{
		SpawnedTab->SetContent(ViewportWidget.ToSharedRef());
	}
	return SpawnedTab;
}

TSharedRef<SDockTab> FAssetEditor_XBlueprint::SpawnDetailsTab(const FSpawnTabArgs& Args)
{
	ensure(Args.GetTabId() == XBlueprintPropertySID);

	return SNew(SDockTab)
	.Icon(FAppStyle::GetBrush("LevelEditor.Tabs.Details"))
	.Label(LOCTEXT("Details_Title", "Property"))
	[
		PropertyWidget.ToSharedRef()
	];
}

TSharedRef<SDockTab> FAssetEditor_XBlueprint::SpawnSearchTab(const FSpawnTabArgs& Args)
{
	ensure(Args.GetTabId() == SearchTabSID);
	RETURN_IF_FALSE(SearchComponent, SNew(SDockTab))

	bSearchTabVisible = true;

	return SNew(SDockTab)
	.Label(LOCTEXT("Search_Title", "Search"))
	[
		SearchComponent->GetSearchWidget()
	];
}

void FAssetEditor_XBlueprint::CreateInternalWidgets()
{
	ViewportWidget = CreateViewportWidget();

	FDetailsViewArgs Args;
	Args.bHideSelectionTip = true;
	Args.NotifyHook = this;

	FPropertyEditorModule& PropertyModule = FModuleManager::LoadModuleChecked<FPropertyEditorModule>("PropertyEditor");
	PropertyWidget = PropertyModule.CreateDetailView(Args);
	PropertyWidget->SetObject(CurrentEditingXBlueprint);
	PropertyWidget->OnFinishedChangingProperties().AddSP(this, &FAssetEditor_XBlueprint::OnFinishedChangingProperties);
}

TSharedRef<SGraphEditor> FAssetEditor_XBlueprint::CreateViewportWidget()
{
	FGraphAppearanceInfo AppearanceInfo;
	AppearanceInfo.CornerText = BlueprintName;

	CreateCommandList();

	SGraphEditor::FGraphEditorEvents InEvents;
	InEvents.OnSelectionChanged = SGraphEditor::FOnSelectionChanged::CreateSP(this, &FAssetEditor_XBlueprint::OnSelectedNodesChanged);
	InEvents.OnTextCommitted = FOnNodeTextCommitted::CreateSP(this, &FAssetEditor_XBlueprint::OnNodeTitleCommitted);

	return SNew(SGraphEditor)
		.AdditionalCommands(GraphEditorCommands)
		.IsEditable(true)
		.Appearance(AppearanceInfo)
		.GraphToEdit(CurrentEditingXBlueprint->Graph)
		.GraphEvents(InEvents)
		.AutoExpandActionMenu(true)
		.ShowGraphStateOverlay(false);
}

void FAssetEditor_XBlueprint::BindCommands()
{
	ToolkitCommands->MapAction(FEditorCommands_XBlueprint::Get().CreateComment,
		FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CreateCommentNode),
		FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanCreateCommentNode)
	);
	ToolkitCommands->MapAction(FEditorCommands_XBlueprint::Get().ToggleSearchTab,
		FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::ToggleSearchTab),
		FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanToggleSearchTab)
	);
}

void FAssetEditor_XBlueprint::CreateGraph()
{
	RETURN_IF_FALSE(CurrentEditingXBlueprint)
	UXGraph* Graph = CurrentEditingXBlueprint->Graph;
	if (!Graph)
	{
		Graph = Cast<UXGraph>(FBlueprintEditorUtils::CreateNewGraph(CurrentEditingXBlueprint, NAME_None, GetGraphClass(), GetSchemaClass()));
		RETURN_IF_FALSE(Graph);

		CurrentEditingXBlueprint->Graph = Graph;
		Graph->bAllowDeletion = false;

		const UEdGraphSchema* Schema = Graph->GetSchema();
		RETURN_IF_FALSE(Schema);
		Schema->CreateDefaultNodesForGraph(*Graph);
	}
}

void FAssetEditor_XBlueprint::CreateCommandList()
{
	RETURN_IF_FALSE(!GraphEditorCommands.IsValid());

	GraphEditorCommands = MakeShared<FUICommandList>();
	// Can't use CreateSP here because derived editor are already implementing TSharedFromThis<FAssetEditorToolkit>
	// however it should be safe, since commands are being used only within this editor
	// if it ever crashes, this function will have to go away and be reimplemented in each derived class

	GraphEditorCommands->MapAction(FGenericCommands::Get().SelectAll,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::SelectAllNodes),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanSelectAllNodes)
	);

	GraphEditorCommands->MapAction(FGenericCommands::Get().Delete,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::DeleteSelectedNodes),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanDeleteNodes)
	);

	GraphEditorCommands->MapAction(FGenericCommands::Get().Copy,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CopySelectedNodes),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanCopyNodes)
	);

	GraphEditorCommands->MapAction(FGenericCommands::Get().Cut,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CutSelectedNodes),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanCutNodes)
	);

	GraphEditorCommands->MapAction(FGenericCommands::Get().Paste,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::PasteNodes),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanPasteNodes)
	);

	GraphEditorCommands->MapAction(FGenericCommands::Get().Duplicate,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::DuplicateNodes),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanDuplicateNodes)
	);

	GraphEditorCommands->MapAction(FGenericCommands::Get().Rename,
	                               FExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::OnRenameNode),
	                               FCanExecuteAction::CreateSP(this, &FAssetEditor_XBlueprint::CanRenameNodes)
	);
}

TSharedPtr<SGraphEditor> FAssetEditor_XBlueprint::GetCurrentGraphEditor() const
{
	return ViewportWidget;
}

FGraphPanelSelectionSet FAssetEditor_XBlueprint::GetSelectedNodes() const
{
	FGraphPanelSelectionSet CurrentSelection;
	TSharedPtr<SGraphEditor> CurrentGraphEditor{ GetCurrentGraphEditor() };
	if (CurrentGraphEditor.IsValid())
	{
		CurrentSelection = CurrentGraphEditor->GetSelectedNodes();
		for (FGraphPanelSelectionSet::TIterator SelectedIter(CurrentSelection); SelectedIter; ++SelectedIter)
		{
			if (UEdGraphNode* const Node{ Cast<UEdGraphNode>(*SelectedIter) };
				!Node)
			{
				SelectedIter.RemoveCurrent();
			}
		}
	}
	return CurrentSelection;
}

void FAssetEditor_XBlueprint::SelectNode(const FString& NodeName)
{
	if (!CurrentEditingXBlueprint || !CurrentEditingXBlueprint->Graph)
	{
		return;
	}

	TSharedPtr<SGraphEditor> CurrentGraphEditor{ GetCurrentGraphEditor() };
	if (!CurrentGraphEditor.IsValid())
	{
		return;
	}

	FString NodeTitle{ NodeName };
	TObjectPtr<UEdGraphNode>* const CenterOnNode{ CurrentEditingXBlueprint->Graph->Nodes.FindByPredicate([this, &NodeTitle](UEdGraphNode* Node)
	{
		UEditorXNode* XNode{ Cast<UEditorXNode>(Node) };
		if (!XNode)
		{
			return false;
		}

		ConvertNodeSIDToTitle(NodeTitle);

		if (NodeTitle == XNode->GetTitle().ToString())
		{
			return true;
		}

		if (const auto* const ConditionNode{ Cast<UConditionEditorQuestNode>(XNode) })
		{
			const FString PinDelim{ TEXT("_Pin_") };
			const int32 PinDelimIndex{ NodeTitle.Find(PinDelim) };
			if (PinDelimIndex == INDEX_NONE)
			{
				return false;
			}

			const FString LinkedNodeName{ NodeTitle.Left(PinDelimIndex) };
			const int32 PinIndex{ FCString::Atoi(*NodeTitle.Right(PinDelimIndex + PinDelim.Len())) };

			const UEdGraphPin* const OutputPin{ ConditionNode->GetDefaultOutputPin() };
			if (!OutputPin || !OutputPin->LinkedTo.IsValidIndex(PinIndex))
			{
				return false;
			}

			if (const UEditorXNode* const LinkedNode{ Cast<UEditorXNode>(OutputPin->LinkedTo[PinIndex]->GetOwningNode()) };
				LinkedNode && LinkedNode->GetTitle().ToString() == LinkedNodeName)
			{
				return true;
			}
		}

		return false;
	})};

	if (CenterOnNode)
	{
		CurrentGraphEditor->JumpToNode(*CenterOnNode);
	}
	else
	{
		UE_LOG(LogTemp, Display, TEXT("Node wasn't found: %s"), *NodeName);
	}
}

void FAssetEditor_XBlueprint::SelectAllNodes()
{
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (CurrentGraphEditor.IsValid())
	{
		CurrentGraphEditor->SelectAllNodes();
	}
}

bool FAssetEditor_XBlueprint::CanSelectAllNodes() const
{
	return true;
}

void FAssetEditor_XBlueprint::DeleteSelectedNodes()
{
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (!CurrentGraphEditor.IsValid())
	{
		return;
	}

	const FScopedTransaction Transaction(FGenericCommands::Get().Delete->GetDescription());

	CurrentGraphEditor->GetCurrentGraph()->Modify();

	const FGraphPanelSelectionSet SelectedNodes = CurrentGraphEditor->GetSelectedNodes();
	CurrentGraphEditor->ClearSelectionSet();

	for (FGraphPanelSelectionSet::TConstIterator NodeIt(SelectedNodes); NodeIt; ++NodeIt)
	{
		UEdGraphNode* EdNode = Cast<UEdGraphNode>(*NodeIt);
		if (!EdNode || !EdNode->CanUserDeleteNode())
		{
			continue;
		}

		EdNode->Modify();

		const UEdGraphSchema* Schema = EdNode->GetSchema();
		if (Schema)
		{
			Schema->BreakNodeLinks(*EdNode);
		}

		EdNode->DestroyNode();
	}
}

bool FAssetEditor_XBlueprint::CanDeleteNodes() const
{
	// If any of the nodes can be deleted then we should allow deleting
	const FGraphPanelSelectionSet SelectedNodes = GetSelectedNodes();
	for (FGraphPanelSelectionSet::TConstIterator SelectedIter(SelectedNodes); SelectedIter; ++SelectedIter)
	{
		UEdGraphNode* Node = Cast<UEdGraphNode>(*SelectedIter);
		if (Node && Node->CanUserDeleteNode())
		{
			return true;
		}
	}
	return false;
}

void FAssetEditor_XBlueprint::DeleteSelectedDuplicatableNodes()
{
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (!CurrentGraphEditor.IsValid())
	{
		return;
	}

	const FGraphPanelSelectionSet OldSelectedNodes = CurrentGraphEditor->GetSelectedNodes();
	CurrentGraphEditor->ClearSelectionSet();

	for (FGraphPanelSelectionSet::TConstIterator SelectedIter(OldSelectedNodes); SelectedIter; ++SelectedIter)
	{
		UEdGraphNode* Node = Cast<UEdGraphNode>(*SelectedIter);
		if (Node && Node->CanDuplicateNode())
		{
			CurrentGraphEditor->SetNodeSelection(Node, true);
		}
	}

	// Delete the duplicatable nodes
	DeleteSelectedNodes();

	CurrentGraphEditor->ClearSelectionSet();

	for (FGraphPanelSelectionSet::TConstIterator SelectedIter(OldSelectedNodes); SelectedIter; ++SelectedIter)
	{
		if (UEdGraphNode* Node = Cast<UEdGraphNode>(*SelectedIter))
		{
			CurrentGraphEditor->SetNodeSelection(Node, true);
		}
	}
}

void FAssetEditor_XBlueprint::CutSelectedNodes()
{
	CopySelectedNodes();
	DeleteSelectedDuplicatableNodes();
}

bool FAssetEditor_XBlueprint::CanCutNodes() const
{
	return CanCopyNodes() && CanDeleteNodes();
}

FString FAssetEditor_XBlueprint::GetNodesText(const FGraphPanelSelectionSet& SelectedNodes)
{
	for (auto* const Node : SelectedNodes)
	{
		if (UEdGraphNode* const GraphNode{ Cast<UEdGraphNode>(Node) })
		{
			GraphNode->PrepareForCopying();
		}
	}

	FString ExportedText;
	FEdGraphUtilities::ExportNodesToText(SelectedNodes, ExportedText);
	return ExportedText;
}

void FAssetEditor_XBlueprint::CopySelectedNodes()
{
	// Export the selected nodes and place the text on the clipboard
	FGraphPanelSelectionSet SelectedNodes{ GetSelectedNodes() };
	FPlatformApplicationMisc::ClipboardCopy(*GetNodesText(SelectedNodes));
}

bool FAssetEditor_XBlueprint::CanCopyNodes() const
{
	// If any of the nodes can be duplicated then we should allow copying
	const FGraphPanelSelectionSet SelectedNodes = GetSelectedNodes();
	for (FGraphPanelSelectionSet::TConstIterator SelectedIter(SelectedNodes); SelectedIter; ++SelectedIter)
	{
		UEdGraphNode* Node = Cast<UEdGraphNode>(*SelectedIter);
		if (Node && Node->CanDuplicateNode())
		{
			return true;
		}
	}
	return false;
}

void FAssetEditor_XBlueprint::PasteNodes()
{
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (CurrentGraphEditor.IsValid())
	{
		PasteNodesHere(CurrentGraphEditor->GetPasteLocation());
	}
}

void FAssetEditor_XBlueprint::PasteNodesHere(const FVector2D& Location)
{
	// Find the graph editor with focus
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (!CurrentGraphEditor.IsValid())
	{
		return;
	}
	// Select the newly pasted stuff
	if (UEdGraph* EdGraph = CurrentGraphEditor->GetCurrentGraph())
	{
		const FScopedTransaction Transaction(FGenericCommands::Get().Paste->GetDescription());
		EdGraph->Modify();

		// Clear the selection set (newly pasted stuff will be selected)
		CurrentGraphEditor->ClearSelectionSet();

		// Grab the text to paste from the clipboard.
		FString TextToImport;
		FPlatformApplicationMisc::ClipboardPaste(TextToImport);

		// Import the nodes
		TSet<UEdGraphNode*> PastedNodes{ ImportNodesFromText(EdGraph, TextToImport) };
		if (PastedNodes.IsEmpty())
		{
			return;
		}

		//Average position of nodes so we can move them while still maintaining relative distances to each other
		FVector2D AvgNodePosition(0.0f, 0.0f);

		for (TSet<UEdGraphNode*>::TIterator It(PastedNodes); It; ++It)
		{
			UEdGraphNode* Node = *It;
			AvgNodePosition.X += Node->NodePosX;
			AvgNodePosition.Y += Node->NodePosY;
		}

		float InvNumNodes = 1.0f / float(PastedNodes.Num());
		AvgNodePosition.X *= InvNumNodes;
		AvgNodePosition.Y *= InvNumNodes;

		for (TSet<UEdGraphNode*>::TIterator It(PastedNodes); It; ++It)
		{
			UEdGraphNode* Node = *It;
			CurrentGraphEditor->SetNodeSelection(Node, true);

			Node->NodePosX = (Node->NodePosX - AvgNodePosition.X) + Location.X;
			Node->NodePosY = (Node->NodePosY - AvgNodePosition.Y) + Location.Y;

			Node->SnapToGrid(16);

			// Give new node a different Guid from the old one
			Node->CreateNewGuid();
		}

		// Update UI
		CurrentGraphEditor->NotifyGraphChanged();

		UObject* GraphOwner = EdGraph->GetOuter();
		if (GraphOwner)
		{
			GraphOwner->PostEditChange();
			GraphOwner->MarkPackageDirty();
		}
	}
}

bool FAssetEditor_XBlueprint::CanPasteNodes() const
{
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (!CurrentGraphEditor.IsValid())
	{
		return false;
	}

	FString ClipboardContent;
	FPlatformApplicationMisc::ClipboardPaste(ClipboardContent);

	return FEdGraphUtilities::CanImportNodesFromText(CurrentGraphEditor->GetCurrentGraph(), ClipboardContent);
}

void FAssetEditor_XBlueprint::DuplicateNodes()
{
	CopySelectedNodes();
	PasteNodes();
}

bool FAssetEditor_XBlueprint::CanDuplicateNodes() const
{
	return CanCopyNodes() && CanPasteNodes();
}

void FAssetEditor_XBlueprint::OnRenameNode()
{
	TSharedPtr<SGraphEditor> CurrentGraphEditor = GetCurrentGraphEditor();
	if (CurrentGraphEditor.IsValid())
	{
		const FGraphPanelSelectionSet SelectedNodes = GetSelectedNodes();
		for (FGraphPanelSelectionSet::TConstIterator NodeIt(SelectedNodes); NodeIt; ++NodeIt)
		{
			UEdGraphNode* SelectedNode = Cast<UEdGraphNode>(*NodeIt);
			if (SelectedNode && SelectedNode->bCanRenameNode)
			{
				CurrentGraphEditor->IsNodeTitleVisible(SelectedNode, true);
				break;
			}
		}
	}
}

bool FAssetEditor_XBlueprint::CanRenameNodes() const
{
	FGraphPanelSelectionSet SelectedNodes = GetSelectedNodes();
	if (SelectedNodes.Num() == 1)
	{
		for (UObject* SelectedNode : SelectedNodes)
		{
			UEdGraphNode* XNode = Cast<UEdGraphNode>(SelectedNode);
			if (XNode)
			{
				return XNode->GetCanRenameNode();
			}
		}
	}
	return false;
}

void FAssetEditor_XBlueprint::CreateCommentNode()
{
	RETURN_IF_FALSE(CurrentEditingXBlueprint)
	RETURN_IF_FALSE(CurrentEditingXBlueprint->Graph)
	TSharedPtr<SGraphEditor> GraphEditor = GetCurrentGraphEditor();
	RETURN_IF_FALSE(GraphEditor)
	FAssetSchemaAction_XBlueprint_NewNode::SpawnNodeFromTemplate<UEditorCommentNode>(CurrentEditingXBlueprint->Graph, NewObject<UEditorCommentNode>(), GraphEditor->GetPasteLocation());
}

bool FAssetEditor_XBlueprint::CanCreateCommentNode() const
{
	return true;
}

void FAssetEditor_XBlueprint::ToggleSearchTab()
{
	if (TabManager)
	{
		const bool bSearchTabDesiredVisible = !bSearchTabVisible;
		TSharedPtr<SDockTab> SearchTab = TabManager->TryInvokeTab(SearchTabSID);
		if (!bSearchTabDesiredVisible)
		{
			SearchTab->RequestCloseTab();
		}
		bSearchTabVisible = bSearchTabDesiredVisible;
	}
}

bool FAssetEditor_XBlueprint::CanToggleSearchTab() const
{
	return true;
}

void FAssetEditor_XBlueprint::OnSelectedNodesChanged(const TSet<UObject*>& NewSelection)
{
	const TArray<TWeakObjectPtr<UObject>>& OldSelection = PropertyWidget->GetSelectedObjects();
	for (const TWeakObjectPtr<UObject>& SelectedObject : OldSelection)
	{
		if (UEditorXNode* SelectedNode = Cast<UEditorXNode>(SelectedObject.Get()))
		{
			if (!NewSelection.Contains(SelectedNode))
			{
				SelectedNode->OnDeselected();
			}
		}
	}

	TArray<UObject*> FilteredNewSelection;
	for (UObject* SelectionEntry : NewSelection)
	{
		if (SelectionEntry)
		{
			FilteredNewSelection.Add(SelectionEntry);
			if (UEditorXNode* SelectedNode = Cast<UEditorXNode>(SelectionEntry))
			{
				if (!OldSelection.Contains(SelectedNode))
				{
					SelectedNode->OnSelected();
				}
			}
		}
	}

	if (FilteredNewSelection.Num() == 0)
	{
		PropertyWidget->SetObject(CurrentEditingXBlueprint);
	}
	else
	{
		PropertyWidget->SetObjects(FilteredNewSelection);
	}
}

void FAssetEditor_XBlueprint::OnNodeTitleCommitted(const FText& NewText, ETextCommit::Type CommitInfo, UEdGraphNode* NodeBeingChanged)
{
	if (NodeBeingChanged)
	{
		const FScopedTransaction Transaction(NSLOCTEXT("RenameNode", "RenameNode", "Rename Node"));
		NodeBeingChanged->Modify();
		NodeBeingChanged->OnRenameNode(NewText.ToString());
	}
}

void FAssetEditor_XBlueprint::OnFinishedChangingProperties(const FPropertyChangedEvent& PropertyChangedEvent)
{
	if (CurrentEditingXBlueprint && CurrentEditingXBlueprint->Graph)
	{
		const UEdGraphSchema* Schema = CurrentEditingXBlueprint->Graph->GetSchema();
		RETURN_IF_FALSE(Schema)
		Schema->ForceVisualizationCacheClear();
	}
}

TSet<UEdGraphNode*> FAssetEditor_XBlueprint::ImportNodesFromText(UEdGraph* EdGraph, const FString& TextToImport)
{
	FXGraphTextFactory Factory(EdGraph, GetNodeClass());
	Factory.ProcessBuffer(EdGraph, RF_Transactional, TextToImport);
	FEdGraphUtilities::PostProcessPastedNodes(Factory.SpawnedNodes);
	UEdGraphPin::ResolveAllPinReferences();
	return Factory.SpawnedNodes;
}
```