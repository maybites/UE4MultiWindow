# UE4 Plugin for Scene Capture rendering to separate window

[Tutorial](https://geodesic.tech/ue4-plugin-for-scene-capture-rendering-to-separate-window/)

[![UE4 Plugin for Scene Capture rendering to separate window](https://user-images.githubusercontent.com/6189196/53569672-039f0680-3b65-11e9-98c2-395a4d47af64.PNG)](https://www.youtube.com/watch?v=8tyCK89ceMM)

Unreal engine is a really powerful Engine, but when you try to work with custom rendering, multi-window, or your own shaders and GPU draw calls everything becomes more complicated.

The main reason for over-complication is the custom-made and HUGE UE4 rendering pipeline. Essentially UE4 made something called a “slate framework” as its own GUI, which works through 3D GPU acceleration but is meant for both 2D and 3D Graphics. I won’t say custom rendering is really hard in UE4, but a huge amount of rendering interfaces, modules, and dependencies makes it more complicated.

Today I want to describe and share with you the plugin code for MultiWindow ScreenCapture Rendering; https://github.com/Batname/UE4MultiWindow. I’ve had a challenge with building in custom rendering and custom window creation in the past and now I want to share my knowledge.

We made a small plugin which is focused only on rendering a custom SceneRenderingComponent texture to a Custom Window which works during editing and playing in UE4 Editor.

Essentially this is an Editor plugin which creates a new button in Editor UI, and when you click the Play Scene button opens a new custom window and renders a ScreenCapture texture from the scene to this window.

Alright, lets take a look into the code and go through the logic step by step.

> The plugin is called PlayScene and logic execution starts in PlayScene.cpp

```
void FPlaySceneModule::StartupModule()
{
  // Initialize play button ui style
	FPlaySceneStyle::Initialize();
	FPlaySceneStyle::ReloadTextures();

	// Register play capture commands
	FPlaySceneCommands::Register();
	PluginCommands = MakeShareable(new FUICommandList);

	// Add play capture button command
	PluginCommands->MapAction(
		FPlaySceneCommands::Get().OpenPluginWindow,
		FExecuteAction::CreateRaw(this, &FPlaySceneModule::PluginButtonClicked),
		FCanExecuteAction());

	FLevelEditorModule& LevelEditorModule = FModuleManager::LoadModuleChecked<FLevelEditorModule>("LevelEditor");

	// Add play capture button to editor
	{
		TSharedPtr<FExtender> ToolbarExtender = MakeShareable(new FExtender);
		ToolbarExtender->AddToolBarExtension("Settings", EExtensionHook::After, PluginCommands, FToolBarExtensionDelegate::CreateRaw(this, &FPlaySceneModule::AddToolbarExtension));

		LevelEditorModule.GetToolBarExtensibilityManager()->AddExtender(ToolbarExtender);
	}
}
```


In StartupModule() we execute logic for Editor PlayScene button styles, register UI commands and all handlers

> When we click PlayScene Button it executes the function PluginButtonClicked()

```
void FPlaySceneModule::PluginButtonClicked()
{
  // Init layCapture Window
  FPlaySceneSlate::Initialize();
}
```

And it creates a window and starts rendering, let’s move to file PlaySceneSlate.cpp

> Here is how we create the window and assign a viewport

```
FPlaySceneSlate::FPlaySceneSlate()
: PlaySceneWindowWidth(1280)
, PlaySceneWindowHeight(720)
{
  // Create SWindow
	PlaySceneWindow = SNew(SWindow)
		.Title(FText::FromString("PlaySceneWindow"))
		.ScreenPosition(FVector2D(0, 0))
		.ClientSize(FVector2D(PlaySceneWindowWidth, PlaySceneWindowHeight))
		.AutoCenter(EAutoCenter::PreferredWorkArea)
		.UseOSWindowBorder(true)
		.SaneWindowPlacement(false)
		.SizingRule(ESizingRule::UserSized);

	FSlateApplication::Get().AddWindow(PlaySceneWindow.ToSharedRef());

	// Assign window events delegator
	InDelegate.BindRaw(this, &FPlaySceneSlate::OnWindowClosed);
	PlaySceneWindow->SetOnWindowClosed(InDelegate);

	// Create and assign viewport to window
	PlaySceneViewport = SNew(SPlaySceneViewport);
	PlaySceneWindow->SetContent(PlaySceneViewport.ToSharedRef());
}
```

> In our custom Viewport we need to create Viewport Client, Scene Viewport and SetViewportInterface to Slate Viewport

```
void SPlaySceneViewport::Construct(const FArguments&amp; InArgs)
{
  // Create Viewport Widget
  Viewport = SNew(SViewport)
  .IsEnabled(true)
  .EnableGammaCorrection(false)
  .ShowEffectWhenDisabled(false)
  .EnableBlending(true)
  .ToolTip(SNew(SToolTip).Text(FText::FromString("SPlaySceneViewport")));

  // Create Viewport Client
  PlaySceneViewportClient = MakeShareable(new FPlaySceneViewportClient());

  // Create Scene Viewport
  SceneViewport = MakeShareable(new FSceneViewport(PlaySceneViewportClient.Get(), Viewport));

  // Assign SceneViewport to Viewport widget. It needed for rendering
  Viewport->SetViewportInterface(SceneViewport.ToSharedRef());

  // Assing Viewport widget for our custom PlayScene Viewport
  this->ChildSlot
  [
  Viewport.ToSharedRef()
  ];
}
```

> And then comes the Draw function inside FPlaySceneViewportClient, which is executed each frame and issues the draw call for our Texture from PCSceneCaptureComponent2D

```
void FPlaySceneViewportClient::Draw(FViewport * Viewport, FCanvas * Canvas)
{
  // Clear entire canvas
  Canvas->Clear(FLinearColor::Black);

  // Draw SceneCaptureComponent texture to entire canvas
  auto TextRenderTarget2D = IPlayScene::Get().GetTextureRenderTarget2D();
  if (TextRenderTarget2D.IsValid() && TextRenderTarget2D->Resource != nullptr)
  {
  FCanvasTileItem TileItem(FVector2D(0, 0), TextRenderTarget2D-&gt;Resource,
  FVector2D(Viewport->GetRenderTargetTexture()->GetSizeX(), Viewport->GetRenderTargetTexture()>GetSizeY()),
  FLinearColor::White);
  TileItem.BlendMode = ESimpleElementBlendMode::SE_BLEND_Opaque;
  Canvas->DrawItem(TileItem);
  }
}
```


Now you can open up your scene. Go to Content Browser > View Options > Show Plugin Content > enable and lookup the PCSceneCaptureComponent2D to add to your scene.

Create a Render Target Asset and link it to the newly created PCSceneCaptureComponent2D.

Off you go.

Ways to extend this plugin:

I would like to be able to configure the window within the editor before it is opened. So more sophistication has to be added: one way to do it:

https://forums.unrealengine.com/t/how-to-make-a-dockable-tab-in-a-plugin-also-hamads-plugin-corrected-unreal-4-7-0/20524
