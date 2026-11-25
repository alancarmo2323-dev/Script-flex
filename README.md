-- BLACK HUB by Flex
-- Universal Hub para Delta (Estilo Sidebar + Cards com Descrição e Toggle)

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local VirtualUser = game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- ==========================================
-- CONFIGURAÇÕES GERAIS E ESTADO
-- ==========================================
local HubConfig = {
	Title = "BLACK HUB",
	Creator = "Flex",
	Subtitle = "by Flex • Delta Universal",
	ToggleKey = Enum.KeyCode.RightControl,
	Theme = {
		Background = Color3.fromRGB(20, 21, 25),
		Sidebar = Color3.fromRGB(16, 17, 20),
		Topbar = Color3.fromRGB(24, 25, 30),
		Card = Color3.fromRGB(28, 30, 36),
		TabActive = Color3.fromRGB(36, 39, 48),
		Accent = Color3.fromRGB(88, 101, 242),
		ToggleOff = Color3.fromRGB(52, 55, 65),
		TextPrimary = Color3.fromRGB(240, 242, 248),
		TextSecondary = Color3.fromRGB(145, 150, 165),
		Border = Color3.fromRGB(40, 43, 52),
	}
}

local State = {
	WalkSpeedEnabled = false,
	WalkSpeedValue = 30,
	JumpEnabled = false,
	JumpPowerValue = 80,
	InfiniteJump = false,
	NoClip = false,
	FlyEnabled = false,
	FlySpeed = 65,
	ClickTeleport = false,

	ESPEnabled = false,
	ESPNames = true,
	Fullbright = false,
	NoFog = false,
	CustomFOVEnabled = false,
	FOVValue = 90,

	AimbotEnabled = false,
	AimbotSmoothness = 0.2,
	AimbotTargetPart = "Head",
	ShowFOVCircle = false,
	FOVRadius = 140,
	HitboxExpander = false,
	HitboxSize = 8,

	AntiAFK = true,
	FPSBoost = false,
}

local Connections = {}
local ESPObjects = {}
local OriginalHitboxes = {}
local OriginalLighting = {
	Brightness = Lighting.Brightness,
	FogEnd = Lighting.FogEnd,
	FogStart = Lighting.FogStart,
	GlobalShadows = Lighting.GlobalShadows,
}

-- ==========================================
-- FUNÇÕES DE AMBIENTE DE EXECUTOR
-- ==========================================
local function getGuiParent()
	if gethui then
		local ok, hui = pcall(gethui)
		if ok and hui then
			return hui
		end
	end
	local okCore, coreGui = pcall(function()
		return game:GetService("CoreGui")
	end)
	if okCore and coreGui then
		local testOk = pcall(function()
			local f = Instance.new("Folder")
			f.Parent = coreGui
			f:Destroy()
		end)
		if testOk then
			return coreGui
		end
	end
	return LocalPlayer:WaitForChild("PlayerGui")
end

local function safeClipboard(text)
	if setclipboard then
		pcall(setclipboard, tostring(text))
		return true
	elseif toclipboard then
		pcall(toclipboard, tostring(text))
		return true
	end
	return false
end

local function getHumanoid()
	local char = LocalPlayer.Character
	return char and char:FindFirstChildOfClass("Humanoid")
end

local function getRootPart()
	local char = LocalPlayer.Character
	return char and char:FindFirstChild("HumanoidRootPart")
end

local function tween(instance, info, props)
	local anim = TweenService:Create(instance, info, props)
	anim:Play()
	return anim
end

local function addCorner(parent, radius)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius or 8)
	c.Parent = parent
	return c
end

local function addStroke(parent, color, thickness)
	local s = Instance.new("UIStroke")
	s.Color = color or HubConfig.Theme.Border
	s.Thickness = thickness or 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = parent
	return s
end

local function addPadding(parent, top, bottom, left, right)
	local p = Instance.new("UIPadding")
	p.PaddingTop = UDim.new(0, top or 8)
	p.PaddingBottom = UDim.new(0, bottom or 8)
	p.PaddingLeft = UDim.new(0, left or 10)
	p.PaddingRight = UDim.new(0, right or 10)
	p.Parent = parent
	return p
end

local function makeDraggable(dragHandle, targetWindow)
	local dragging = false
	local dragInput, dragStart, startPos

	dragHandle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = targetWindow.Position

			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
				end
			end)
		end
	end)

	dragHandle.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			dragInput = input
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if input == dragInput and dragging then
			local delta = input.Position - dragStart
			targetWindow.Position = UDim2.new(
				startPos.X.Scale,
				startPos.X.Offset + delta.X,
				startPos.Y.Scale,
				startPos.Y.Offset + delta.Y
			)
		end
	end)
end

-- ==========================================
-- CRIAÇÃO DA INTERFACE
-- ==========================================
local GuiParent = getGuiParent()
local oldGui = GuiParent:FindFirstChild("BlackHub_Flex")
if oldGui then
	oldGui:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BlackHub_Flex"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
pcall(function()
	ScreenGui.Parent = GuiParent
end)
if not ScreenGui.Parent then
	ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

-- Botão flutuante para abrir/fechar fácil no Delta
local ReopenButton = Instance.new("TextButton")
ReopenButton.Name = "ReopenButton"
ReopenButton.Size = UDim2.fromOffset(126, 36)
ReopenButton.Position = UDim2.new(0, 14, 0.5, -18)
ReopenButton.BackgroundColor3 = HubConfig.Theme.Topbar
ReopenButton.Text = "  BLACK HUB"
ReopenButton.TextColor3 = HubConfig.Theme.TextPrimary
ReopenButton.Font = Enum.Font.GothamBold
ReopenButton.TextSize = 13
ReopenButton.TextXAlignment = Enum.TextXAlignment.Left
ReopenButton.Visible = false
ReopenButton.Parent = ScreenGui
addCorner(ReopenButton, 8)
addStroke(ReopenButton, HubConfig.Theme.Accent, 1.2)
makeDraggable(ReopenButton, ReopenButton)

local ReopenDot = Instance.new("Frame")
ReopenDot.Size = UDim2.fromOffset(8, 8)
ReopenDot.Position = UDim2.new(1, -16, 0.5, -4)
ReopenDot.BackgroundColor3 = HubConfig.Theme.Accent
ReopenDot.Parent = ReopenButton
addCorner(ReopenDot, 999)

-- Janela principal
local MainWindow = Instance.new("Frame")
MainWindow.Name = "MainWindow"
MainWindow.Size = UDim2.fromOffset(640, 390)
MainWindow.Position = UDim2.new(0.5, -320, 0.5, -195)
MainWindow.BackgroundColor3 = HubConfig.Theme.Background
MainWindow.ClipsDescendants = true
MainWindow.Parent = ScreenGui
addCorner(MainWindow, 10)
addStroke(MainWindow, HubConfig.Theme.Border, 1.2)

local UIScale = Instance.new("UIScale")
local vp = Camera.ViewportSize
local scaleFactor = math.clamp(math.min(vp.X / 740, vp.Y / 460), 0.65, 1)
UIScale.Scale = scaleFactor
UIScale.Parent = MainWindow

-- Topbar
local Topbar = Instance.new("Frame")
Topbar.Name = "Topbar"
Topbar.Size = UDim2.new(1, 0, 0, 38)
Topbar.BackgroundColor3 = HubConfig.Theme.Topbar
Topbar.BorderSizePixel = 0
Topbar.Parent = MainWindow
makeDraggable(Topbar, MainWindow)

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(0.3, 0, 1, 0)
TitleLabel.Position = UDim2.fromOffset(14, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = HubConfig.Title
TitleLabel.TextColor3 = HubConfig.Theme.TextPrimary
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextSize = 14
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.Parent = Topbar

local SubtitleLabel = Instance.new("TextLabel")
SubtitleLabel.Size = UDim2.new(0.45, 0, 1, 0)
SubtitleLabel.Position = UDim2.fromOffset(106, 0)
SubtitleLabel.BackgroundTransparency = 1
SubtitleLabel.Text = HubConfig.Subtitle
SubtitleLabel.TextColor3 = HubConfig.Theme.TextSecondary
SubtitleLabel.Font = Enum.Font.Gotham
SubtitleLabel.TextSize = 11
SubtitleLabel.TextXAlignment = Enum.TextXAlignment.Left
SubtitleLabel.Parent = Topbar

local MinButton = Instance.new("TextButton")
MinButton.Size = UDim2.fromOffset(32, 26)
MinButton.Position = UDim2.new(1, -74, 0.5, -13)
MinButton.BackgroundColor3 = Color3.fromRGB(34, 36, 43)
MinButton.Text = "—"
MinButton.TextColor3 = HubConfig.Theme.TextSecondary
MinButton.Font = Enum.Font.GothamBold
MinButton.TextSize = 13
MinButton.Parent = Topbar
addCorner(MinButton, 6)

local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.fromOffset(32, 26)
CloseButton.Position = UDim2.new(1, -38, 0.5, -13)
CloseButton.BackgroundColor3 = Color3.fromRGB(34, 36, 43)
CloseButton.Text = "✕"
CloseButton.TextColor3 = HubConfig.Theme.TextSecondary
CloseButton.Font = Enum.Font.GothamBold
CloseButton.TextSize = 13
CloseButton.Parent = Topbar
addCorner(CloseButton, 6)

local isMinimized = false
local normalSize = MainWindow.Size

MinButton.MouseButton1Click:Connect(function()
	isMinimized = not isMinimized
	tween(MainWindow, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {
		Size = isMinimized and UDim2.fromOffset(640, 38) or normalSize
	})
end)

CloseButton.MouseButton1Click:Connect(function()
	MainWindow.Visible = false
	ReopenButton.Visible = true
end)

ReopenButton.MouseButton1Click:Connect(function()
	MainWindow.Visible = true
	ReopenButton.Visible = false
end)

UserInputService.InputBegan:Connect(function(input, processed)
	if not processed and input.KeyCode == HubConfig.ToggleKey then
		local nextState = not MainWindow.Visible
		MainWindow.Visible = nextState
		ReopenButton.Visible = not nextState
	end
end)

-- Corpo
local BodyContainer = Instance.new("Frame")
BodyContainer.Size = UDim2.new(1, 0, 1, -38)
BodyContainer.Position = UDim2.fromOffset(0, 38)
BodyContainer.BackgroundTransparency = 1
BodyContainer.Parent = MainWindow

local Sidebar = Instance.new("ScrollingFrame")
Sidebar.Name = "Sidebar"
Sidebar.Size = UDim2.new(0, 160, 1, 0)
Sidebar.BackgroundColor3 = HubConfig.Theme.Sidebar
Sidebar.BorderSizePixel = 0
Sidebar.ScrollBarThickness = 2
Sidebar.CanvasSize = UDim2.fromOffset(0, 0)
Sidebar.AutomaticCanvasSize = Enum.AutomaticSize.Y
Sidebar.Parent = BodyContainer
addPadding(Sidebar, 10, 10, 8, 8)

local SidebarLayout = Instance.new("UIListLayout")
SidebarLayout.Padding = UDim.new(0, 5)
SidebarLayout.SortOrder = Enum.SortOrder.LayoutOrder
SidebarLayout.Parent = Sidebar

local Divider = Instance.new("Frame")
Divider.Size = UDim2.new(0, 1, 1, 0)
Divider.Position = UDim2.fromOffset(160, 0)
Divider.BackgroundColor3 = HubConfig.Theme.Border
Divider.BorderSizePixel = 0
Divider.Parent = BodyContainer

local ContentArea = Instance.new("Frame")
ContentArea.Name = "ContentArea"
ContentArea.Size = UDim2.new(1, -161, 1, 0)
ContentArea.Position = UDim2.fromOffset(161, 0)
ContentArea.BackgroundColor3 = HubConfig.Theme.Background
ContentArea.BorderSizePixel = 0
ContentArea.Parent = BodyContainer

-- ==========================================
-- COMPONENTES: ABAS, SEÇÕES, TOGGLES, SLIDERS E BOTÕES
-- ==========================================
local Tabs = {}
local ActiveTabButton = nil
local ActiveTabPage = nil

local function createTab(name, iconText)
	local index = #Tabs + 1

	local TabBtn = Instance.new("TextButton")
	TabBtn.Name = "Tab_" .. name
	TabBtn.Size = UDim2.new(1, 0, 0, 34)
	TabBtn.BackgroundColor3 = HubConfig.Theme.Sidebar
	TabBtn.BackgroundTransparency = 1
	TabBtn.Text = ""
	TabBtn.LayoutOrder = index
	TabBtn.Parent = Sidebar
	addCorner(TabBtn, 7)

	local Indicator = Instance.new("Frame")
	Indicator.Size = UDim2.new(0, 3, 0.6, 0)
	Indicator.Position = UDim2.new(0, 2, 0.2, 0)
	Indicator.BackgroundColor3 = HubConfig.Theme.Accent
	Indicator.BackgroundTransparency = 1
	Indicator.BorderSizePixel = 0
	Indicator.Parent = TabBtn
	addCorner(Indicator, 3)

	local IconLabel = Instance.new("TextLabel")
	IconLabel.Size = UDim2.fromOffset(22, 22)
	IconLabel.Position = UDim2.new(0, 10, 0.5, -11)
	IconLabel.BackgroundTransparency = 1
	IconLabel.Text = iconText
	IconLabel.TextColor3 = HubConfig.Theme.TextSecondary
	IconLabel.Font = Enum.Font.GothamBold
	IconLabel.TextSize = 13
	IconLabel.Parent = TabBtn

	local Label = Instance.new("TextLabel")
	Label.Size = UDim2.new(1, -38, 1, 0)
	Label.Position = UDim2.fromOffset(36, 0)
	Label.BackgroundTransparency = 1
	Label.Text = name
	Label.TextColor3 = HubConfig.Theme.TextSecondary
	Label.Font = Enum.Font.GothamMedium
	Label.TextSize = 12.5
	Label.TextXAlignment = Enum.TextXAlignment.Left
	Label.Parent = TabBtn

	local Page = Instance.new("ScrollingFrame")
	Page.Name = "Page_" .. name
	Page.Size = UDim2.fromScale(1, 1)
	Page.BackgroundTransparency = 1
	Page.BorderSizePixel = 0
	Page.ScrollBarThickness = 4
	Page.ScrollBarImageColor3 = HubConfig.Theme.Border
	Page.CanvasSize = UDim2.fromOffset(0, 0)
	Page.AutomaticCanvasSize = Enum.AutomaticSize.Y
	Page.Visible = false
	Page.Parent = ContentArea
	addPadding(Page, 12, 16, 14, 14)

	local PageLayout = Instance.new("UIListLayout")
	PageLayout.Padding = UDim.new(0, 8)
	PageLayout.SortOrder = Enum.SortOrder.LayoutOrder
	PageLayout.Parent = Page

	local function selectTab()
		if ActiveTabButton and ActiveTabPage then
			ActiveTabButton.Button.BackgroundTransparency = 1
			ActiveTabButton.Indicator.BackgroundTransparency = 1
			ActiveTabButton.Label.TextColor3 = HubConfig.Theme.TextSecondary
			ActiveTabButton.IconLabel.TextColor3 = HubConfig.Theme.TextSecondary
			ActiveTabPage.Visible = false
		end

		TabBtn.BackgroundTransparency = 0
		TabBtn.BackgroundColor3 = HubConfig.Theme.TabActive
		Indicator.BackgroundTransparency = 0
		Label.TextColor3 = HubConfig.Theme.TextPrimary
		IconLabel.TextColor3 = HubConfig.Theme.TextPrimary
		Page.Visible = true

		ActiveTabButton = {
			Button = TabBtn,
			Indicator = Indicator,
			Label = Label,
			IconLabel = IconLabel,
		}
		ActiveTabPage = Page
	end

	TabBtn.MouseButton1Click:Connect(selectTab)
	table.insert(Tabs, { Select = selectTab })

	if index == 1 then
		selectTab()
	end

	return Page
end

local function addSection(parent, titleText)
	local SectionLabel = Instance.new("TextLabel")
	SectionLabel.Size = UDim2.new(1, 0, 0, 24)
	SectionLabel.BackgroundTransparency = 1
	SectionLabel.Text = titleText
	SectionLabel.TextColor3 = HubConfig.Theme.TextPrimary
	SectionLabel.Font = Enum.Font.GothamBold
	SectionLabel.TextSize = 14
	SectionLabel.TextXAlignment = Enum.TextXAlignment.Left
	SectionLabel.Parent = parent
	return SectionLabel
end

local function addToggle(parent, title, description, defaultValue, callback)
	local state = defaultValue or false

	local Card = Instance.new("Frame")
	Card.Size = UDim2.new(1, 0, 0, 52)
	Card.BackgroundColor3 = HubConfig.Theme.Card
	Card.Parent = parent
	addCorner(Card, 8)
	addStroke(Card, HubConfig.Theme.Border, 1)

	local TitleTxt = Instance.new("TextLabel")
	TitleTxt.Size = UDim2.new(1, -76, 0, 20)
	TitleTxt.Position = UDim2.fromOffset(12, 7)
	TitleTxt.BackgroundTransparency = 1
	TitleTxt.Text = title
	TitleTxt.TextColor3 = HubConfig.Theme.TextPrimary
	TitleTxt.Font = Enum.Font.GothamSemiBold
	TitleTxt.TextSize = 13
	TitleTxt.TextXAlignment = Enum.TextXAlignment.Left
	TitleTxt.Parent = Card

	local DescTxt = Instance.new("TextLabel")
	DescTxt.Size = UDim2.new(1, -76, 0, 16)
	DescTxt.Position = UDim2.fromOffset(12, 27)
	DescTxt.BackgroundTransparency = 1
	DescTxt.Text = description
	DescTxt.TextColor3 = HubConfig.Theme.TextSecondary
	DescTxt.Font = Enum.Font.Gotham
	DescTxt.TextSize = 11
	DescTxt.TextXAlignment = Enum.TextXAlignment.Left
	DescTxt.TextTruncate = Enum.TextTruncate.AtEnd
	DescTxt.Parent = Card

	local SwitchBg = Instance.new("TextButton")
	SwitchBg.Size = UDim2.fromOffset(42, 22)
	SwitchBg.Position = UDim2.new(1, -54, 0.5, -11)
	SwitchBg.BackgroundColor3 = state and HubConfig.Theme.Accent or HubConfig.Theme.ToggleOff
	SwitchBg.Text = ""
	SwitchBg.AutoButtonColor = false
	SwitchBg.Parent = Card
	addCorner(SwitchBg, 999)

	local Knob = Instance.new("Frame")
	Knob.Size = UDim2.fromOffset(16, 16)
	Knob.Position = state and UDim2.fromOffset(23, 3) or UDim2.fromOffset(3, 3)
	Knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
	Knob.Parent = SwitchBg
	addCorner(Knob, 999)

	local function setState(newVal, silent)
		state = newVal
		tween(SwitchBg, TweenInfo.new(0.16, Enum.EasingStyle.Quad), {
			BackgroundColor3 = state and HubConfig.Theme.Accent or HubConfig.Theme.ToggleOff
		})
		tween(Knob, TweenInfo.new(0.16, Enum.EasingStyle.Quad), {
			Position = state and UDim2.fromOffset(23, 3) or UDim2.fromOffset(3, 3)
		})
		if not silent and callback then
			pcall(callback, state)
		end
	end

	SwitchBg.MouseButton1Click:Connect(function()
		setState(not state, false)
	end)

	Card.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			setState(not state, false)
		end
	end)

	if state and callback then
		task.defer(function()
			pcall(callback, true)
		end)
	end
end

local function addSlider(parent, title, description, minVal, maxVal, defaultVal, callback)
	local currentVal = math.clamp(defaultVal or minVal, minVal, maxVal)

	local Card = Instance.new("Frame")
	Card.Size = UDim2.new(1, 0, 0, 62)
	Card.BackgroundColor3 = HubConfig.Theme.Card
	Card.Parent = parent
	addCorner(Card, 8)
	addStroke(Card, HubConfig.Theme.Border, 1)

	local TitleTxt = Instance.new("TextLabel")
	TitleTxt.Size = UDim2.new(0.7, 0, 0, 18)
	TitleTxt.Position = UDim2.fromOffset(12, 6)
	TitleTxt.BackgroundTransparency = 1
	TitleTxt.Text = title
	TitleTxt.TextColor3 = HubConfig.Theme.TextPrimary
	TitleTxt.Font = Enum.Font.GothamSemiBold
	TitleTxt.TextSize = 12.5
	TitleTxt.TextXAlignment = Enum.TextXAlignment.Left
	TitleTxt.Parent = Card

	local ValueLabel = Instance.new("TextLabel")
	ValueLabel.Size = UDim2.new(0.25, -12, 0, 18)
	ValueLabel.Position = UDim2.new(0.75, 0, 0, 6)
	ValueLabel.BackgroundTransparency = 1
	ValueLabel.Text = tostring(currentVal)
	ValueLabel.TextColor3 = HubConfig.Theme.Accent
	ValueLabel.Font = Enum.Font.GothamBold
	ValueLabel.TextSize = 12.5
	ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
	ValueLabel.Parent = Card

	local DescTxt = Instance.new("TextLabel")
	DescTxt.Size = UDim2.new(1, -24, 0, 14)
	DescTxt.Position = UDim2.fromOffset(12, 23)
	DescTxt.BackgroundTransparency = 1
	DescTxt.Text = description
	DescTxt.TextColor3 = HubConfig.Theme.TextSecondary
	DescTxt.Font = Enum.Font.Gotham
	DescTxt.TextSize = 10.5
	DescTxt.TextXAlignment = Enum.TextXAlignment.Left
	DescTxt.Parent = Card

	local BarBg = Instance.new("Frame")
	BarBg.Size = UDim2.new(1, -24, 0, 6)
	BarBg.Position = UDim2.fromOffset(12, 45)
	BarBg.BackgroundColor3 = HubConfig.Theme.ToggleOff
	BarBg.Parent = Card
	addCorner(BarBg, 999)

	local ratio = (currentVal - minVal) / math.max(1, (maxVal - minVal))
	local Fill = Instance.new("Frame")
	Fill.Size = UDim2.fromScale(ratio, 1)
	Fill.BackgroundColor3 = HubConfig.Theme.Accent
	Fill.Parent = BarBg
	addCorner(Fill, 999)

	local sliding = false
	local function updateFromInput(inputPosX)
		local rel = math.clamp((inputPosX - BarBg.AbsolutePosition.X) / math.max(1, BarBg.AbsoluteSize.X), 0, 1)
		local val = math.floor(minVal + (maxVal - minVal) * rel + 0.5)
		currentVal = val
		Fill.Size = UDim2.fromScale(rel, 1)
		ValueLabel.Text = tostring(val)
		if callback then
			pcall(callback, val)
		end
	end

	BarBg.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = true
			updateFromInput(input.Position.X)
		end
	end)

	UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = false
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			updateFromInput(input.Positi
