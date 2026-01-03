--[[
    SCRIPT: TP 100% (FIX LỖI KHÔNG CHẠY)
    * FIX: Tăng cường khả năng đọc giá trị của CircularProgressBar (sử dụng GetAttribute và ValueObject).
    * Ngưỡng Teleport = 1.0 (100%).
    * Tự động cầm thảm khi tiến trình bắt đầu.
]]

local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local StarterGui = game:GetService("StarterGui")

-- CẤU HÌNH CỤ THỂ CHO PROGRESS
local PROGRESS_CONTAINER_PATH = "ProximityPrompts"
-- Đường dẫn đầy đủ tới CircularProgressBar
local PROGRESS_ELEMENT_PATH_SUFFIX = "Prompt.Frame.InputFrame.Frame.CircularProgressBar" 
local TP_THRESHOLD = 1.00 -- Ngưỡng Teleport (100%)
local CARPET_NAMES = {"Carpet", "Thảm", "Magic Carpet", "MagicCarpet", "Thảm Bay", "CarpetTool"} 

local SavedPosition = nil 
local IsMonitoring = false 
local IsCoolingDown = false 
local CurrentProgressConnection = nil 

-- --- UI REFERENCES (Giữ nguyên) ---
local MainFrame = nil 
local StatusLabel = nil
local ProgressFill = nil
local ToggleButtonInstance = nil 
local Dragging = false
local DragOffset = nil

-- --- FUNCTIONS (LOGIC CHUNG) ---

local function Notify(title, message, duration)
    pcall(function()
        StarterGui:SetCore("SendNotification", {Title = title, Text = message, Duration = duration or 3})
    end)
end

local function UpdateStatus(text)
    if StatusLabel and StatusLabel.Parent then StatusLabel.Text = text end
end

local function UpdateProgressBar(progress)
    if ProgressFill and ProgressFill.Parent then
        ProgressFill.Size = UDim2.new(progress, 0, 1, 0)
        ProgressFill.BackgroundColor3 = Color3.fromHSV(0.33 * progress, 1, 1) 
        ProgressFill:FindFirstChild("ProgressText").Text = string.format("%d%%", math.floor(progress * 100))
    end
end

local function ToggleButtonText(enabled)
    if ToggleButtonInstance and ToggleButtonInstance.Parent then
        ToggleButtonInstance.BackgroundColor3 = enabled and Color3.fromRGB(0, 150, 0) or Color3.fromRGB(200, 0, 0)
        ToggleButtonInstance.Text = enabled and "STOP MONITORING" or "START MONITORING (USER TRIGGERED)"
    end
end

-- HÀM TỰ ĐỘNG CẦM THẢM
local function EquipCarpet()
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    local character = LocalPlayer.Character
    
    if not backpack or not character then return end
    
    for _, toolName in ipairs(CARPET_NAMES) do
        if character:FindFirstChild(toolName) then return end
    end

    for _, tool in ipairs(backpack:GetChildren()) do
        if tool:IsA("Tool") then
            for _, nameKey in ipairs(CARPET_NAMES) do
                if string.find(tool.Name:lower(), nameKey:lower()) then
                    tool.Parent = character
                    return
                end
            end
        end
    end
end

local function TeleportBack()
    if not SavedPosition then return end
    local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    if CurrentProgressConnection then
        pcall(function() CurrentProgressConnection:Disconnect() end)
        CurrentProgressConnection = nil
    end

    pcall(function() hrp.CFrame = CFrame.new(SavedPosition) end)
    
    UpdateStatus("TP THÀNH CÔNG: Đã Teleport về Base!")
    Notify("TP TRIGGER", "Đã cướp xong và TP về Base.", 2)
    
    IsCoolingDown = true
    UpdateProgressBar(0) 
    task.wait(3) 
    IsCoolingDown = false
    
    if IsMonitoring then StartMonitoring() end
end

-- HÀM TÌM KIẾM ĐỐI TƯỢNG THEO CHUỖI ĐƯỜNG DẪN CỤ THỂ
local function FindElementByPath(root, path)
    local parts = path:split('.')
    local current = root
    for _, part in ipairs(parts) do
        if not current then return nil end
        current = current:FindFirstChild(part)
    end
    return current
end

-- HÀM ĐỌC GIÁ TRỊ TỐI ƯU (FIX LỖI KHÔNG CHẠY)
local function GetCurrentProgress(element)
    -- Cách 1: Thử đọc thuộc tính 'Progress' (phổ biến)
    local success, val = pcall(function() return element.Progress end)
    if success and type(val) == "number" then return val end
    
    -- Cách 2: Thử đọc Attribute "Progress" (rất phổ biến cho CircularProgressBar)
    local attr = element:GetAttribute("Progress")
    if attr and type(attr) == "number" then return attr end

    -- Cách 3: Thử tìm ValueObject con tên là "Progress"
    local childVal = element:FindFirstChild("Progress")
    if childVal and childVal:IsA("ValueBase") then return childVal.Value end
    
    -- Cách 4: Đọc Size.X.Scale (Phòng trường hợp là thanh ngang)
    if element:IsA("GuiObject") then return element.Size.X.Scale end
    
    return 0
end

-- --- MONITORING LOGIC (HEARTBEAT ONLY) ---

local function MonitorProgressFrame(progressElement)
    if CurrentProgressConnection then
        pcall(function() CurrentProgressConnection:Disconnect() end)
        CurrentProgressConnection = nil
    end

    local function CheckProgress()
        local currentProgress = GetCurrentProgress(progressElement)
        
        -- Xử lý giá trị (Chuyển 0-100 thành 0-1)
        if type(currentProgress) ~= "number" then currentProgress = 0 end
        if currentProgress < 0 then currentProgress = 0 end
        if currentProgress > 1 and currentProgress <= 100 then currentProgress = currentProgress / 100 end 
        
        UpdateProgressBar(currentProgress)
        UpdateStatus(string.format("Monitoring: Progress at %d%% | TP @ 100%%", math.floor(currentProgress * 100)))

        if currentProgress > 0 and not IsCoolingDown then
            -- 1. Tự động cầm thảm khi tiến trình bắt đầu
            if currentProgress > 0.01 then
                EquipCarpet()
            end

            -- 2. Teleport khi đạt ngưỡng 100%
            if currentProgress >= TP_THRESHOLD then
                TeleportBack()
            end
        end
    end
    
    CurrentProgressConnection = RunService.Heartbeat:Connect(CheckProgress)
    Notify("MONITOR START", "Đã phát hiện tiến trình cướp nhà. Đang giám sát...", 1)
end

local function FindAndMonitorElement()
    if not IsMonitoring or IsCoolingDown then return end
    
    local ProximityGui = PlayerGui:FindFirstChild(PROGRESS_CONTAINER_PATH)
    if not ProximityGui then return end
    
    -- Tìm phần tử tiến trình bằng đường dẫn chi tiết
    local targetElement = FindElementByPath(ProximityGui, PROGRESS_ELEMENT_PATH_SUFFIX)
    
    if targetElement then 
        MonitorProgressFrame(targetElement)
        return true
    end
    return false
end

local function StartMonitoring()
    if IsMonitoring then return end
    if not SavedPosition then Notify("LỖỖI", "Vui lòng đặt Waypoint trước.", 3) return end
    
    IsMonitoring = true
    ToggleButtonText(true)
    UpdateStatus("Monitoring Activated. Start the steal manually now.")

    local ProximityGui = PlayerGui:FindFirstChild(PROGRESS_CONTAINER_PATH)
    if ProximityGui then
        ProximityGui.ChildAdded:Connect(function()
            task.wait(0.2) 
            FindAndMonitorElement()
        end)
    end
    
    if not FindAndMonitorElement() then
        UpdateStatus("Chờ GUI cướp xuất hiện...")
        Notify("READY", "Đã sẵn sàng. Vui lòng tự kích hoạt cướp để hiện thanh tiến trình.", 2)
    end
end

local function StopMonitoring()
    IsMonitoring = false
    ToggleButtonText(false)
    UpdateStatus("Monitoring Stopped.")
    UpdateProgressBar(0)
    if CurrentProgressConnection then
        pcall(function() CurrentProgressConnection:Disconnect() end)
        CurrentProgressConnection = nil
    end
end

local function ToggleMonitoring()
    if IsMonitoring then
        StopMonitoring()
    else
        StartMonitoring()
    end
end

local function SaveCurrentPosition()
    local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then Notify("LỖỖI", "Nhân vật chưa tải.", 3) return end

    SavedPosition = hrp.Position
    Notify("THÀNH CÔNG", "Đã đặt Waypoint.", 3)
    UpdateStatus("Waypoint Set! Ready to monitor.")
end

-- --- LOGIC KÉO MENU (DRAGGABLE UI) ---
local function SetupDraggableUI(frame)
    local inputService = game:GetService("UserInputService")

    local function OnInputBegan(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            if frame.Parent and input.Target == frame or input.Target.Parent == frame then
                Dragging = true
                DragOffset = Vector2.new(input.Position.X, input.Position.Y) - Vector2.new(frame.AbsolutePosition.X, frame.AbsolutePosition.Y)
            end
        end
    end

    local function OnInputEnded(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            Dragging = false
        end
    end
    
    RunService.RenderStepped:Connect(function()
        if Dragging then
             local touchPosition = inputService:GetTouchStates()[1] and inputService:GetTouchStates()[1].Position or inputService:GetMouseLocation()
             local newPos = Vector2.new(touchPosition.X, touchPosition.Y) - DragOffset
             frame.Position = UDim2.new(0, newPos.X, 0, newPos.Y)
        end
    end)
    
    inputService.InputBegan:Connect(OnInputBegan)
    inputService.InputEnded:Connect(OnInputEnded)
end

-- --- TẠO UI (MENU) ---
local function CreateTigyMenuUI()
    local success, results = pcall(function()
        if PlayerGui:FindFirstChild("ProgressTriggerUI") then PlayerGui.ProgressTriggerUI:Destroy() end

        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Name = "ProgressTriggerUI"
        ScreenGui.Parent = PlayerGui
        
        -- --- MENU FRAME ---
        local MainFrameInstance = Instance.new("Frame")
        MainFrameInstance.Size = UDim2.new(0, 250, 0, 160) 
        MainFrameInstance.Position = UDim2.new(0.05, 0, 0.5, 0) 
        MainFrameInstance.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
        MainFrameInstance.BorderSizePixel = 1
        MainFrameInstance.BorderColor3 = Color3.fromRGB(200, 100, 0)
        MainFrameInstance.Parent = ScreenGui
        MainFrame = MainFrameInstance
        SetupDraggableUI(MainFrameInstance) 

        local TitleLabel = Instance.new("TextLabel")
        TitleLabel.Text = "TP CƯỚP NHÀ (FIX LỖI CHẠY)" 
        TitleLabel.Font = Enum.Font.GothamBold
        TitleLabel.TextColor3 = Color3.fromRGB(255, 165, 0)
        TitleLabel.TextSize = 13
        TitleLabel.Size = UDim2.new(1, 0, 0, 25)
        TitleLabel.Position = UDim2.new(0, 0, 0, 0)
        TitleLabel.BackgroundTransparency = 1
        TitleLabel.Parent = MainFrameInstance
        
        local StatusLabelInstance = Instance.new("TextLabel")
        StatusLabelInstance.Name = "StatusLabelInstance"
        StatusLabelInstance.Text = "Script Loaded."
        StatusLabelInstance.Font = Enum.Font.SourceSans
        StatusLabelInstance.TextColor3 = Color3.fromRGB(170, 170, 170)
        StatusLabelInstance.TextSize = 12
        StatusLabelInstance.Size = UDim2.new(0.9, 0, 0, 15)
        StatusLabelInstance.Position = UDim2.new(0.05, 0, 0.2, 0)
        StatusLabelInstance.BackgroundTransparency = 1
        StatusLabelInstance.TextXAlignment = Enum.TextXAlignment.Left
        StatusLabelInstance.Parent = MainFrameInstance
        StatusLabel = StatusLabelInstance

        local ProgressFrameInstance = Instance.new("Frame")
        ProgressFrameInstance.Name = "ProgressFrame"
        ProgressFrameInstance.Size = UDim2.new(0.9, 0, 0, 15)
        ProgressFrameInstance.Position = UDim2.new(0.05, 0, 0.35, 0)
        ProgressFrameInstance.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        ProgressFrameInstance.BorderSizePixel = 0
        ProgressFrameInstance.Parent = MainFrameInstance

        local ProgressFillInstance = Instance.new("Frame")
        ProgressFillInstance.Name = "ProgressFill"
        ProgressFillInstance.Size = UDim2.new(0, 0, 1, 0)
        ProgressFillInstance.BackgroundColor3 = Color3.fromRGB(0, 200, 0)
        ProgressFillInstance.BorderSizePixel = 0
        ProgressFillInstance.Parent = ProgressFrameInstance
        ProgressFill = ProgressFillInstance

        local ProgressText = Instance.new("TextLabel")
        ProgressText.Name = "ProgressText"
        ProgressText.Text = "0%"
        ProgressText.Font = Enum.Font.SourceSansBold
        ProgressText.TextColor3 = Color3.fromRGB(255, 255, 255)
        ProgressText.TextSize = 12
        ProgressText.Size = UDim2.new(1, 0, 1, 0)
        ProgressText.BackgroundTransparency = 1
        ProgressText.Parent = ProgressFillInstance
        
        local SaveButton = Instance.new("TextButton")
        SaveButton.Text = "1. Set Waypoint (Base Về)"
        SaveButton.Font = Enum.Font.SourceSans
        SaveButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        SaveButton.TextSize = 16
        SaveButton.Size = UDim2.new(0.9, 0, 0, 30)
        SaveButton.Position = UDim2.new(0.05, 0, 0.5, 0)
        SaveButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
        SaveButton.BorderSizePixel = 0
        SaveButton.Parent = MainFrameInstance
        SaveButton.MouseButton1Click:Connect(SaveCurrentPosition)
        
        local ToggleButtonInstance_Ref = Instance.new("TextButton")
        ToggleButtonInstance_Ref.Name = "ToggleButton"
        ToggleButtonInstance_Ref.Text = "START MONITORING (USER TRIGGERED)"
        ToggleButtonInstance_Ref.Font = Enum.Font.SourceSans
        ToggleButtonInstance_Ref.TextColor3 = Color3.fromRGB(255, 255, 255)
        ToggleButtonInstance_Ref.TextSize = 16
        ToggleButtonInstance_Ref.Size = UDim2.new(0.9, 0, 0, 30)
        ToggleButtonInstance_Ref.Position = UDim2.new(0.05, 0, 0.75, 0)
        ToggleButtonInstance_Ref.BackgroundColor3 = Color3.fromRGB(0, 150, 0) 
        ToggleButtonInstance_Ref.BorderSizePixel = 0
        ToggleButtonInstance_Ref.Parent = MainFrameInstance
        ToggleButtonInstance = ToggleButtonInstance_Ref
        ToggleButtonInstance.MouseButton1Click:Connect(ToggleMonitoring)
        
        UpdateStatus("Script Loaded. Set Waypoint first.")
        Notify("TP TRIGGER READY", "Đã sẵn sàng.", 3)
    end)
    
    if not success then
         Notify("LỖI UI", "Menu không hiển thị.", 5)
    end
end

pcall(CreateTigyMenuUI)
