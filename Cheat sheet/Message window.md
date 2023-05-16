```cpp
const FText DialogText{ LOCTEXT("Text") };
const FText DialogTitle{ LOCTEXT("Title", "Title") };

EAppReturnType::Type MessageWindowInput{ FMessageDialog::Open(EAppMsgType::YesNo, DialogText, &DialogTitle) };
```