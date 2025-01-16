```cpp
USTRUCT()
struct FAssetSchemaAction_XBlueprint_NewNode : public FEdGraphSchemaAction
{
	GENERATED_BODY();

public:
	FAssetSchemaAction_XBlueprint_NewNode() = default;
	FAssetSchemaAction_XBlueprint_NewNode(const FText& InNodeCategory, const FText& InMenuDesc, const FText& InToolTip, const int32 InGrouping) :
		: FEdGraphSchemaAction(InNodeCategory, InMenuDesc, InToolTip, InGrouping)
	{ }

	UEdGraphNode* PerformAction(UEdGraph* ParentGraph, UEdGraphPin* FromPin, const FVector2D Location, bool bSelectNewNode = true) override;
	void AddReferencedObjects(FReferenceCollector& Collector) override;

	template <typename NodeType>
	static NodeType* SpawnNodeFromTemplate(class UEdGraph* ParentGraph, NodeType* InTemplateNode, const FVector2D Location, bool bSelectNewNode = true)
	{
		FAssetSchemaAction_XBlueprint_NewNode Action;
		Action.NodeTemplate = InTemplateNode;
		return Cast<NodeType>(Action.PerformAction(ParentGraph, nullptr, Location, bSelectNewNode));
	}

	UEdGraphNode* NodeTemplate = nullptr;
};

UCLASS(abstract, MinimalAPI)
class UAssetGraphSchema_XBlueprint : public UEdGraphSchema
{
	GENERATED_BODY()

public:
	void GetBreakLinkToSubMenuActions(UToolMenu* Menu, UEdGraphPin* InGraphPin) const;
	EGraphType GetGraphType(const UEdGraph* TestEdGraph) const override;
	void GetContextMenuActions(UToolMenu* Menu, UGraphNodeContextMenuContext* Context) const override;
 	const FPinConnectionResponse CanCreateConnection(const UEdGraphPin* A, const UEdGraphPin* B) const override;
	FLinearColor GetPinTypeColor(const FEdGraphPinType& PinType) const override;
	bool IsCacheVisualizationOutOfDate(int32 InVisualizationCacheID) const override;
	int32 GetCurrentVisualizationCacheID() const override;

	bool CreateAutomaticConversionNodeAndConnections(UEdGraphPin* A, UEdGraphPin* B) const override;
	void OnPinConnectionDoubleCicked(UEdGraphPin* PinA, UEdGraphPin* PinB, const FVector2D& GraphPosition) const override;

	UEdGraphPin* DropPinOnNode(UEdGraphNode* InTargetNode, const FName& InSourcePinName, const FEdGraphPinType& InSourcePinType, EEdGraphPinDirection InSourcePinDirection) const override;
	bool SupportsDropPinOnNode(UEdGraphNode* InTargetNode, const FEdGraphPinType& InSourcePinType, EEdGraphPinDirection InSourcePinDirection, FText& OutErrorMessage) const override;

	void ForceVisualizationCacheClear() const override;

	void BreakPinLinksMenuAction(const FToolMenuContext& InContext) const;

protected:
	TMap<FString, FString> NodeTitlesCorrespondence;

	void AddNodeTypeToContextMenu(FGraphContextMenuBuilder& ContextMenuBuilder, TSet<TSubclassOf<UEdGraphNode>>& CreatableNodeTypes, UClass* NodeType, const FText& Tooltip, const FText& Category) const;
	void AddChildNodeTypesToContextMenu(FGraphContextMenuBuilder& ContextMenuBuilder, TSet<TSubclassOf<UEdGraphNode>>& CreatableNodeTypes, UClass* ParentClass, const FText& Tooltip, const FText& Category) const;
	
private:
	mutable int32 CurrentCacheRefreshID = 0;

	FText GenerateNodeUIName(FString Title) const;
};
```

```cpp
UEdGraphNode* FAssetSchemaAction_XBlueprint_NewNode::PerformAction(UEdGraph* ParentGraph, UEdGraphPin* FromPin, const FVector2D Location, bool bSelectNewNode)
{
	RETURN_IF_FALSE(NodeTemplate, nullptr);
	RETURN_IF_FALSE(ParentGraph, nullptr);

	const FScopedTransaction Transaction(LOCTEXT("XBlueprintEditorNewNode", "X Blueprint Editor: New Node"));
	ParentGraph->Modify();
	if (FromPin)
	{
		FromPin->Modify();
	}

	NodeTemplate->Rename(nullptr, ParentGraph);
	ParentGraph->AddNode(NodeTemplate, true, bSelectNewNode);

	NodeTemplate->CreateNewGuid();
	NodeTemplate->PostPlacedNewNode();
	NodeTemplate->AllocateDefaultPins();
	NodeTemplate->AutowireNewNode(FromPin);

	NodeTemplate->NodePosX = Location.X;
	NodeTemplate->NodePosY = Location.Y;

	NodeTemplate->SetFlags(RF_Transactional);

	return NodeTemplate;
}

void FAssetSchemaAction_XBlueprint_NewNode::AddReferencedObjects(FReferenceCollector& Collector)
{
	FEdGraphSchemaAction::AddReferencedObjects(Collector);
	Collector.AddReferencedObject(NodeTemplate);
}

void UAssetGraphSchema_XBlueprint::GetBreakLinkToSubMenuActions(UToolMenu* Menu, UEdGraphPin* InGraphPin) const
{
	RETURN_IF_FALSE(Menu);
	RETURN_IF_FALSE(InGraphPin);

	// Make sure we have a unique name for every entry in the list
	TMap<FString, uint32> LinkTitleCount;

	for (UEdGraphPin* LinkedToPin : InGraphPin->LinkedTo)
	{
		CONTINUE_IF_FALSE(LinkedToPin);
		UEdGraphNode* LinkedToNode = LinkedToPin->GetOwningNode();
		CONTINUE_IF_FALSE(LinkedToNode);

		FString TitleString = LinkedToNode->GetNodeTitle(ENodeTitleType::ListView).ToString();
		FText Title = FText::FromString(TitleString);
		if (!LinkedToPin->PinName.IsNone())
		{
			TitleString = FString::Printf(TEXT("%s (%s)"), *TitleString, *LinkedToPin->PinName.ToString());

			// Add name of connection if possible
			FFormatNamedArguments Args;
			Args.Add(TEXT("NodeTitle"), Title);
			Args.Add(TEXT("PinName"), LinkedToPin->GetDisplayName());
			Title = FText::Format(LOCTEXT("BreakDescPin", "{NodeTitle} ({PinName})"), Args);
		}

		uint32& Count = LinkTitleCount.FindOrAdd(TitleString);

		FText Description;
		FFormatNamedArguments Args;
		Args.Add(TEXT("NodeTitle"), Title);
		Args.Add(TEXT("NumberOfNodes"), Count);

		if (Count == 0)
		{
			Description = FText::Format(LOCTEXT("BreakDesc", "Break link to {NodeTitle}"), Args);
		}
		else
		{
			Description = FText::Format(LOCTEXT("BreakDescMulti", "Break link to {NodeTitle} ({NumberOfNodes})"), Args);
		}
		++Count;

		FToolMenuSection& Section = Menu->FindOrAddSection("XBlueprintAssetGraphSchemaPinActions");
		Section.AddMenuEntry(NAME_None, Description, Description, FSlateIcon(), FUIAction(
			                     FExecuteAction::CreateUObject(this, &UAssetGraphSchema_XBlueprint::BreakSinglePinLink, InGraphPin, LinkedToPin)));
	}
}

EGraphType UAssetGraphSchema_XBlueprint::GetGraphType(const UEdGraph* TestEdGraph) const
{
	return GT_Ubergraph;
}

void UAssetGraphSchema_XBlueprint::GetContextMenuActions(UToolMenu* Menu, UGraphNodeContextMenuContext* Context) const
{
	Super::GetContextMenuActions(Menu, Context);

	RETURN_IF_FALSE(Menu);
	RETURN_IF_FALSE(Context);

	if (UEdGraphPin* InGraphPin = const_cast<UEdGraphPin*>(Context->Pin))
	{
		RETURN_IF_FALSE(InGraphPin)
		for (int32 i = InGraphPin->LinkedTo.Num() - 1; i >= 0; i--)
		{
			if (!InGraphPin->LinkedTo[i])
			{
				InGraphPin->LinkedTo.RemoveAt(i);
			}
		}

		FToolMenuSection& Section = Menu->AddSection("XBlueprintAssetGraphSchemaNodeActions", LOCTEXT("PinActionsMenuHeader", "Pin Actions"));
		// Break pin links
		if (InGraphPin->LinkedTo.Num() > 0)
		{
			// Break all pin links
			FToolUIAction BreakPinLinksAction;
			BreakPinLinksAction.ExecuteAction = FToolMenuExecuteAction::CreateUObject(this, &UAssetGraphSchema_XBlueprint::BreakPinLinksMenuAction);

			TSharedPtr<FUICommandInfo> BreakPinLinksCommand = FGraphEditorCommands::Get().BreakPinLinks;
			Section.AddMenuEntry(
				BreakPinLinksCommand->GetCommandName(),
				BreakPinLinksCommand->GetLabel(),
				BreakPinLinksCommand->GetDescription(),
				BreakPinLinksCommand->GetIcon(),
				BreakPinLinksAction
			);
		}
		// add sub menu for break link to
		if (InGraphPin->LinkedTo.Num() > 1)
		{
			Section.AddSubMenu("BreakLinkTo", LOCTEXT("BreakLinkTo", "Break Link To..."), LOCTEXT("BreakSpecificLinks", "Break a specific link..."),
			                   FNewToolMenuDelegate::CreateUObject(this, &UAssetGraphSchema_XBlueprint::GetBreakLinkToSubMenuActions, InGraphPin));
		}
		else
		{
			GetBreakLinkToSubMenuActions(Menu, InGraphPin);
		}
	}
	else if (const UEdGraphNode* InGraphNode = Context->Node)
	{
		FToolMenuSection& Section = Menu->AddSection("XBlueprintAssetGraphSchemaNodeActions", LOCTEXT("NodeActionsMenuHeader", "Node Actions"));
		Section.AddMenuEntry(FGenericCommands::Get().Delete);
		Section.AddMenuEntry(FGenericCommands::Get().Cut);
		Section.AddMenuEntry(FGenericCommands::Get().Copy);
		Section.AddMenuEntry(FGenericCommands::Get().Duplicate);
		Section.AddMenuEntry(FGraphEditorCommands::Get().BreakNodeLinks);
		Section.AddMenuEntry(FEditorCommands_XBlueprint::Get().CreatePreset);
	}
}

const FPinConnectionResponse UAssetGraphSchema_XBlueprint::CanCreateConnection(const UEdGraphPin* A, const UEdGraphPin* B) const
{
	if (A && B)
	{
		// Make sure the pins are not on the same node
		const UEditorXNode* const NodeA = Cast<UEditorXNode>(A->GetOwningNode());
		if (NodeA && NodeA == B->GetOwningNode())
		{
			if (!NodeA->CanConnectToSelf())
			{
				return FPinConnectionResponse(CONNECT_RESPONSE_DISALLOW, LOCTEXT("PinErrorSameNode", "Both are on the same node"));
			}
		}

		// Compare the directions
		if (A->Direction == EGPD_Input && B->Direction == EGPD_Input)
		{
			return FPinConnectionResponse(CONNECT_RESPONSE_DISALLOW, LOCTEXT("PinErrorInput", "Can't connect input node to input node"));
		}
		else if (A->Direction == EGPD_Output && B->Direction == EGPD_Output)
		{
			return FPinConnectionResponse(CONNECT_RESPONSE_DISALLOW, LOCTEXT("PinErrorOutput", "Can't connect output node to output node"));
		}

		return FPinConnectionResponse(CONNECT_RESPONSE_MAKE, LOCTEXT("PinConnect", "Connect nodes"));
	}
	return FPinConnectionResponse(CONNECT_RESPONSE_DISALLOW, LOCTEXT("PinErrorNullNode", "Invalid node"));
}

FLinearColor UAssetGraphSchema_XBlueprint::GetPinTypeColor(const FEdGraphPinType& PinType) const
{
	return FLinearColor::White;
}

bool UAssetGraphSchema_XBlueprint::IsCacheVisualizationOutOfDate(int32 InVisualizationCacheID) const
{
	return CurrentCacheRefreshID != InVisualizationCacheID;
}

int32 UAssetGraphSchema_XBlueprint::GetCurrentVisualizationCacheID() const
{
	return CurrentCacheRefreshID;
}

bool UAssetGraphSchema_XBlueprint::CreateAutomaticConversionNodeAndConnections(UEdGraphPin* A, UEdGraphPin* B) const
{
	// X nodes need no conversion so "everything is correctly converted, return true" situation is equal to A && B pins validity
	return A && B;
}

void UAssetGraphSchema_XBlueprint::OnPinConnectionDoubleCicked(UEdGraphPin* PinA, UEdGraphPin* PinB, const FVector2D& GraphPosition) const
{
	RETURN_IF_FALSE(PinA);
	RETURN_IF_FALSE(PinB);

	const FScopedTransaction Transaction(LOCTEXT("CreateRerouteNodeOnWire", "Create Reroute Node"));

	const FVector2D NodeSpacerSize(42.f, 24.f);
	const FVector2D KnotTopLeft = GraphPosition - NodeSpacerSize * 0.5f;

	// Create a new knot
	UEdGraphNode* PinNode = PinA->GetOwningNode();
	RETURN_IF_FALSE(PinNode);

	UEdGraph* ParentGraph = PinNode->GetGraph();
	RETURN_IF_FALSE(ParentGraph);

	UEditorRerouteNode* NewReroute = FAssetSchemaAction_XBlueprint_NewNode::SpawnNodeFromTemplate<UEditorRerouteNode>(ParentGraph, NewObject<UEditorRerouteNode>(), KnotTopLeft);
	RETURN_IF_FALSE(NewReroute);

	// Move the connections across (only notifying the knot, as the other two didn't really change)
	PinA->BreakLinkTo(PinB);
	PinA->MakeLinkTo(PinA->Direction == EGPD_Output ? NewReroute->GetDefaultInputPin() : NewReroute->GetDefaultOutputPin());
	PinB->MakeLinkTo(PinB->Direction == EGPD_Output ? NewReroute->GetDefaultInputPin() : NewReroute->GetDefaultOutputPin());
}

UEdGraphPin* UAssetGraphSchema_XBlueprint::DropPinOnNode(UEdGraphNode* InTargetNode, const FName& InSourcePinName, const FEdGraphPinType& InSourcePinType, EEdGraphPinDirection InSourcePinDirection) const
{
	UEditorXNode* Node = Cast<UEditorXNode>(InTargetNode);
	RETURN_IF_FALSE(Node, nullptr)

	switch (InSourcePinDirection)
	{
		case EGPD_Input:
		{
			return Node->GetDefaultOutputPin();
		}
		case EGPD_Output:
		{
			return Node->GetDefaultInputPin();
		}
		default:
		{
			ENSURE_NO_ENTRY("");
			return nullptr;
		}
	}
}

bool UAssetGraphSchema_XBlueprint::SupportsDropPinOnNode(UEdGraphNode* InTargetNode, const FEdGraphPinType& InSourcePinType, EEdGraphPinDirection InSourcePinDirection, FText& OutErrorMessage) const
{
	if (UEditorXNode* Node = Cast<UEditorXNode>(InTargetNode))
	{
		switch (InSourcePinDirection)
		{
			case EGPD_Input:
			{
				return Node->GetDefaultOutputPin() != nullptr;
			}
			case EGPD_Output:
			{
				return Node->GetDefaultInputPin() != nullptr;
			}
			default:
			{
				ENSURE_NO_ENTRY("Invalid pin type");
				return false;
			}
		}
	}
	return false;
}

void UAssetGraphSchema_XBlueprint::ForceVisualizationCacheClear() const
{
	++CurrentCacheRefreshID;
}

void UAssetGraphSchema_XBlueprint::BreakPinLinksMenuAction(const FToolMenuContext& InContext) const
{
	UGraphNodeContextMenuContext* NodeContext = InContext.FindContext<UGraphNodeContextMenuContext>();
	if (NodeContext && NodeContext->Pin)
	{
		BreakPinLinks(const_cast<UEdGraphPin&>(*NodeContext->Pin), true);
	}
}

void UAssetGraphSchema_XBlueprint::AddNodeTypeToContextMenu(FGraphContextMenuBuilder& ContextMenuBuilder, TSet<TSubclassOf<UEdGraphNode>>& CreatableNodeTypes, UClass* NodeType, const FText& Tooltip, const FText& Category) const
{
	const FText MenuDescription = GenerateNodeUIName(NodeType->GetName());
	const TSharedPtr<FAssetSchemaAction_XBlueprint_NewNode> NewNodeAction(new FAssetSchemaAction_XBlueprint_NewNode(Category, MenuDescription, Tooltip, 0));
	FStaticConstructObjectParameters ConstructNodeParams(NodeType);
	ConstructNodeParams.Outer = ContextMenuBuilder.OwnerOfTemporaries;
	ConstructNodeParams.Name = NAME_None;
	NewNodeAction->NodeTemplate = Cast<UEdGraphNode>(StaticConstructObject_Internal(ConstructNodeParams));
	ContextMenuBuilder.AddAction(NewNodeAction);
	CreatableNodeTypes.Add(NodeType);
}

void UAssetGraphSchema_XBlueprint::AddChildNodeTypesToContextMenu(FGraphContextMenuBuilder& ContextMenuBuilder, TSet<TSubclassOf<UEdGraphNode>>& CreatableNodeTypes, UClass* ParentClass, const FText& Tooltip, const FText& Category) const
{
	for (TObjectIterator<UClass> NodeType; NodeType; ++NodeType)
	{
		if (NodeType &&
			NodeType->IsChildOf(ParentClass) &&
			!NodeType->HasAnyClassFlags(CLASS_Abstract) &&
			!CreatableNodeTypes.Contains(*NodeType)
		{
			AddNodeTypeToContextMenu(ContextMenuBuilder, CreatableNodeTypes, *NodeType, Tooltip, Category);
		}
	}
}

FText UAssetGraphSchema_XBlueprint::GenerateNodeUIName(FString Title) const
{
	return FText::FromString(Title);
}
```