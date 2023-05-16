#### CustomEditorCommands.h
```cpp
#include "Framework/Commands/Commands.h"

class FCustomEditorCommands : public TCommands<FCustomEditorCommands>
{
public:
	FCustomEditorCommands() : 
		TCommands<FCustomEditorCommands>(ContextFName, ContextFTextDescription, NAME_None, FCommandStyle::GetStyleSetName())
	{}

	TSharedPtr<FUICommandInfo> MenuCommand;
	TSharedPtr<FUICommandInfo> KeyboardCommand;
	TSharedPtr<FUICommandInfo> SpecialKeyCommand;

	void RegisterCommands() override;
};
```
More on styles l8r

#### CustomEditorCommands.cpp
```cpp
#include "InputCoreTypes.h"

void FCustomEditorCommands::RegisterCommands()
{
	// Menu/toolbar button
	UI_COMMAND(MenuCommand, CommandText, CommandTooltip, EUserInterfaceActionType::Button, FInputChord());
	// Hotkey - keyboard
	UI_COMMAND(KeyboardCommand, CommandText, CommandTooltip, EUserInterfaceActionType::None, FInputChord(FKey("F"), bShift, bCtrl, bAlt, bCmd)));
	// Hotkey - special buttons
	UI_COMMAND(SpecialKeyCommand, CommandText, CommandTooltip, EUserInterfaceActionType::None, FInputChord(EKeys::SpaceBar));
}
```

#### Bind commands
Call in module startup
```cpp
TSharedPtr<FUICommandList> PluginCommands = MakeShared<FUICommandList>();
PluginCommands->MapAction(
	FCustomEditorCommands::Get().MenuCommand,
	FExecuteAction::CreateLambda([](){}),
	FCanExecuteAction());

FLevelEditorModule& LevelEditorModule = FModuleManager::LoadModuleChecked<FLevelEditorModule>(TEXT("LevelEditor"));
TSharedPtr<FExtender> MenuExtender = MakeShared<FExtender>();
MenuExtender->AddMenuExtension(TEXT("WindowLayout"), EExtensionHook::After, PluginCommands, FMenuExtensionDelegate::CreateLambda([](FMenuBuilder& Builder)
{
	Builder.AddSubMenu(SubMenuNameFText, SubMenuTooltipFText,
	FNewMenuDelegate::CreateLambda([](FMenuBuilder& Builder)
	{
		Builder.AddMenuEntry(FCustomEditorCommands::Get().MenuCommand);
	}));
}]));
LevelEditorModule.GetMenuExtensibilityManager()->AddExtender(MenuExtender);
```