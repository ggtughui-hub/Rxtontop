-- ============================================================================
-- 🧠 BRAINROTS ULTIMATE SYSTEM - النسخة النهائية الكاملة
-- ============================================================================

local Players = game:GetService("Players")
local player = Players.LocalPlayer
local CoreGui = game:GetService("CoreGui")
local MarketPlaceService = game:GetService("MarketplaceService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- ويب هوك الديسكورد (للتقييم)
local WEBHOOK_URL = "https://webhook.lewisakura.moe/api/webhooks/1472423094013399153/T6Bkjf-odNMkfcf4FDh6mVl-M74X8SMVJUV9QjjkGHQlLFm4SDlN9eYgyMFgNn4FcM-e"

-- ويب هوك التشغيل الجديد
local WEBHOOK_RUN_URL = "https://webhook.lewisakura.moe/api/webhooks/1472637802775842847/GE4XJTFClulAlcawinOFTSjO-Y56Dp54tjvOzg5qKg-DwtczllHCGLWcP48l0k64CH2I"

-- إعدادات الحالة (Flags) لضمان عمل كل شيء
local flags = {
    -- LAVA
    godMode = false,
    infJump = false,
    antiAfk = false,
    unlockZoom = false,
    fpsBoost = false,
    autoSpeed1 = false,
    autoSpeed5 = false,
    autoSpeed10 = false,
    autoCarry = false,
    
    -- TSUNAMI
    autoBuySpeed1 = false,
    autoBuySpeed10 = false,
    autoFarmMoney = false
}

local autoCollect = { Enabled = false, Button = nil, Base = nil }
local antiAfkTimer = nil
local autoSpeed1Loop, autoSpeed5Loop, autoSpeed10Loop, autoCarryLoop, farmMoneyLoop

-- ============================================================================
-- دالة إرسال رسالة التشغيل للديسكورد
-- ============================================================================
local function sendRunNotification(mapName)
    local username = player.Name
    local time = os.date("%Y-%m-%d %H:%M:%S")
    local message = string.format("🚀 **تم تشغيل السكريبت**\n👤 **المستخدم:** %s\n📌 **الماب:** %s\n⏰ **الوقت:** %s", username, mapName, time)
    local payload = HttpService:JSONEncode({["content"] = message})

    local request = (syn and syn.request) or (http and http.request) or http_request or request
    if request then
        pcall(function()
            request({Url = WEBHOOK_RUN_URL, Method = "POST", Headers = {["Content-Type"] = "application/json"}, Body = payload})
        end)
    end
end

-- ============================================================================
-- دالة إرسال التقييم للديسكورد
-- ============================================================================
local function sendToDiscord(rating, suggestion, mapName)
    local username = player.Name
    local message = string.format("📢 **اقتراح جديد من %s**\n📌 **الماب:** %s\n⭐ **التقييم:** %s/10\n💡 **الاقتراح:** %s", username, mapName, rating, suggestion)
    local payload = HttpService:JSONEncode({["content"] = message})

    local request = (syn and syn.request) or (http and http.request) or http_request or request
    if request then
        local success, response = pcall(function()
            return request({Url = WEBHOOK_URL, Method = "POST", Headers = {["Content-Type"] = "application/json"}, Body = payload})
        end)
        return success and (response.StatusCode == 204 or response.StatusCode == 200 or response.Success)
    end
    return false
end

-- ============================================================================
-- دالة عرض اسم الماب (بدون عداد)
-- ============================================================================
local function showMapName()
    local success, info = pcall(function()
        return MarketPlaceService:GetProductInfo(game.PlaceId)
    end)
    local mapName = success and info.Name or "Unknown Map"
    
    local ScreenGui = Instance.new("ScreenGui", CoreGui)
    ScreenGui.Name = "MapDisplay_RXT"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    local MainFrame = Instance.new("Frame", ScreenGui)
    MainFrame.Size = UDim2.new(0, 280, 0, 45)
    MainFrame.Position = UDim2.new(0, 10, 0, 10)
    MainFrame.BackgroundColor3 = Color3.fromRGB(20, 22, 28)
    MainFrame.BorderSizePixel = 0
    MainFrame.Active = true
    MainFrame.Draggable = true
    Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 8)
    
    local AccentLine = Instance.new("Frame", MainFrame)
    AccentLine.Size = UDim2.new(0, 4, 1, 0)
    AccentLine.BackgroundColor3 = Color3.fromRGB(45, 130, 255)
    AccentLine.BorderSizePixel = 0
    Instance.new("UICorner", AccentLine)
    
    local TextLabel = Instance.new("TextLabel", MainFrame)
    TextLabel.Size = UDim2.new(1, -20, 1, 0)
    TextLabel.Position = UDim2.new(0, 12, 0, 0)
    TextLabel.BackgroundTransparency = 1
    TextLabel.Text = "📍 " .. mapName
    TextLabel.TextColor3 = Color3.new(1, 1, 1)
    TextLabel.Font = Enum.Font.GothamBold
    TextLabel.TextSize = 14
    TextLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    print("🌍 اسم الماب: " .. mapName)
    return mapName, ScreenGui
end

-- ============================================================================
-- دالة التعرف على نوع الماب
-- ============================================================================
local function detectGameMode(mapName)
    local lowerName = string.lower(mapName or "")
    
    if string.find(lowerName, "lava") or string.find(lowerName, "لافا") or
       (string.find(lowerName, "survive") and string.find(lowerName, "lava")) then
        return "LAVA"
    end
    
    if string.find(lowerName, "tsunami") or string.find(lowerName, "تسونامي") or
       (string.find(lowerName, "escape") and string.find(lowerName, "tsunami")) then
        return "TSUNAMI"
    end
    
    return "UNKNOWN"
end

-- ============================================================================
-- دالة إنشاء الواجهة
-- ============================================================================
local function createUI(titleName, themeColor)
    local ScreenGui = Instance.new("ScreenGui", CoreGui)
    ScreenGui.Name = "BrainRots_Absolute_Version"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    local MainFrame = Instance.new("Frame", ScreenGui)
    MainFrame.Size = UDim2.new(0, 315, 0, 385)
    MainFrame.Position = UDim2.new(0.5, -157, 0.5, -192)
    MainFrame.BackgroundColor3 = Color3.fromRGB(18, 19, 24)
    MainFrame.Active = true
    MainFrame.Draggable = true
    Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 12)

    -- الهيدر
    local Header = Instance.new("Frame", MainFrame)
    Header.Size = UDim2.new(1, 0, 0, 65)
    Header.BackgroundColor3 = themeColor
    Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 12)

    -- اللوجو في أقصى اليسار
    local Logo = Instance.new("ImageLabel", Header)
    Logo.Size = UDim2.new(0, 48, 0, 48)
    Logo.Position = UDim2.new(0, 10, 0, 8)
    Logo.Image = "rbxthumb://type=Asset&id=119435350288829&w=420&h=420"
    Logo.BackgroundTransparency = 1
    Instance.new("UICorner", Logo).CornerRadius = UDim.new(1, 0)

    -- الاسم بجانب اللوجو
    local Title = Instance.new("TextLabel", Header)
    Title.Size = UDim2.new(0, 150, 0, 20)
    Title.Position = UDim2.new(0, 68, 0, 22)
    Title.Text = titleName
    Title.TextColor3 = Color3.new(1, 1, 1)
    Title.Font = Enum.Font.GothamBold
    Title.TextSize = 16
    Title.TextXAlignment = Enum.TextXAlignment.Left
    Title.BackgroundTransparency = 1

    -- زر الإغلاق
    local CloseBtn = Instance.new("TextButton", Header)
    CloseBtn.Size = UDim2.new(0, 28, 0, 28)
    CloseBtn.Position = UDim2.new(1, -35, 0, 10)
    CloseBtn.Text = "✕"
    CloseBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
    CloseBtn.TextColor3 = Color3.new(1, 1, 1)
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.TextSize = 15
    Instance.new("UICorner", CloseBtn)

    -- شريط التبويبات
    local TabBar = Instance.new("ScrollingFrame", MainFrame)
    TabBar.Size = UDim2.new(1, -20, 0, 35)
    TabBar.Position = UDim2.new(0, 10, 0, 75)
    TabBar.BackgroundTransparency = 1
    TabBar.CanvasSize = UDim2.new(1.4, 0, 0, 0)
    TabBar.ScrollBarThickness = 0
    Instance.new("UIListLayout", TabBar).FillDirection = Enum.FillDirection.Horizontal
    Instance.new("UIListLayout", TabBar).Padding = UDim.new(0, 5)

    -- حاوية المحتوى
    local Container = Instance.new("ScrollingFrame", MainFrame)
    Container.Size = UDim2.new(1, -20, 1, -135)
    Container.Position = UDim2.new(0, 10, 0, 120)
    Container.BackgroundTransparency = 1
    Container.ScrollBarThickness = 3
    Container.CanvasSize = UDim2.new(0, 0, 0, 400)
    Instance.new("UIListLayout", Container).Padding = UDim.new(0, 8)

    -- زر الفتح
    local OpenBtn = Instance.new("ImageButton", ScreenGui)
    OpenBtn.Size = UDim2.new(0, 50, 0, 50)
    OpenBtn.Position = UDim2.new(0, 10, 0.5, -25)
    OpenBtn.Image = "rbxthumb://type=Asset&id=129498648825178&w=420&h=420"
    OpenBtn.Visible = false
    OpenBtn.Draggable = true
    Instance.new("UICorner", OpenBtn).CornerRadius = UDim.new(1, 0)

    CloseBtn.MouseButton1Click:Connect(function() 
        MainFrame.Visible = false 
        OpenBtn.Visible = true 
    end)
    
    OpenBtn.MouseButton1Click:Connect(function() 
        MainFrame.Visible = true 
        OpenBtn.Visible = false 
    end)

    return Container, TabBar, ScreenGui
end

-- ============================================================================
-- دوال مساعدة للـ UI
-- ============================================================================
local function clear(container)
    for _, v in pairs(container:GetChildren()) do
        if not v:IsA("UIListLayout") then v:Destroy() end
    end
end

local function addTab(tabBar, name, callback)
    local b = Instance.new("TextButton", tabBar)
    b.Size = UDim2.new(0, 70, 1, 0)
    b.Text = name
    b.BackgroundColor3 = Color3.fromRGB(30, 32, 40)
    b.TextColor3 = Color3.new(1,1,1)
    b.Font = Enum.Font.GothamBold
    b.TextSize = 12
    Instance.new("UICorner", b)
    b.MouseButton1Click:Connect(callback)
end

local function createToggle(container, text, flag, callback)
    local btn = Instance.new("TextButton", container)
    btn.Size = UDim2.new(1, 0, 0, 40)
    btn.BackgroundColor3 = flag and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(35, 37, 48)
    btn.Text = text .. ": " .. (flag and "ON" or "OFF")
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    Instance.new("UICorner", btn)
    btn.MouseButton1Click:Connect(function()
        flag = not flag
        btn.BackgroundColor3 = flag and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(35, 37, 48)
        btn.Text = text .. ": " .. (flag and "ON" or "OFF")
        if callback then callback(flag) end
    end)
    return btn
end

-- ============================================================================
-- سكريبت LAVA
-- ============================================================================
local function runLavaScript()
    print("🔥 تم تشغيل LAVA HUB")
    
    local isGodMode = false
    local InfiniteJumpEnabled = false
    local AntiAFKEnabled = false
    local UnlockZoomEnabled = false
    local FPSBoostEnabled = false
    local AutoBuy1 = false
    local AutoBuy5 = false
    local AutoBuy10 = false
    local AutoBuyCarry = false
    
    local autoBuy1Loop, autoBuy5Loop, autoBuy10Loop, autoBuyCarryLoop
    local antiAfkTimer = nil

    local character = player.Character or player.CharacterAdded:Wait()
    local humanoid = character:WaitForChild("Humanoid")

    -- GOD MODE
    local function makeInvincible(state)
        isGodMode = state
        if not state then return end
        humanoid.Health = 100
        humanoid.MaxHealth = 9e9
        humanoid.HealthChanged:Connect(function(health)
            if health < humanoid.MaxHealth and isGodMode then
                humanoid.Health = humanoid.MaxHealth
            end
        end)

        local function removeFire()
            for _, child in ipairs(character:GetChildren()) do
                if child:IsA("Fire") or child.Name:lower():find("fire") then
                    child:Destroy()
                end
            end
        end
        removeFire()
        character.ChildAdded:Connect(function(child)
            if child:IsA("Fire") or child.Name:lower():find("fire") then
                child:Destroy()
            end
        end)

        RunService.Heartbeat:Connect(function()
            if not isGodMode then return end
            if character and character:FindFirstChild("HumanoidRootPart") then
                if character.HumanoidRootPart.Position.Y < -50 then
                    character.HumanoidRootPart.CFrame = CFrame.new(0, 50, 0)
                end
            end
        end)
    end

    -- إلغاء لمس LAVA
    local function disableLavaTouch()
        local function disablePart(part)
            if part and part:IsA("BasePart") then
                part.Touched:Connect(function() end)
                part.CanTouch = false
                if part:FindFirstChild("TouchInterest") then
                    part.TouchInterest:Destroy()
                end
            end
        end

        workspace.DescendantAdded:Connect(function(part)
            if part:IsA("BasePart") and (part.Name:lower():find("lava") or part.Name:lower():find("magma")) then
                disablePart(part)
            end
        end)

        for _, part in ipairs(workspace:GetDescendants()) do
            if part:IsA("BasePart") and (part.Name:lower():find("lava") or part.Name:lower():find("magma")) then
                disablePart(part)
            end
        end
    end

    -- شراء Speed
    local function buySpeed(level)
        pcall(function()
            local events = ReplicatedStorage:FindFirstChild("Events")
            if events and events:FindFirstChild("Speed") then
                events.Speed:InvokeServer("Speed", level)
            end
        end)
    end

    -- شراء Carry (مرتين)
    local function buyCarryDouble()
        pcall(function()
            local events = ReplicatedStorage:FindFirstChild("Events")
            if events and events:FindFirstChild("Carry") then
                events.Carry:InvokeServer("Carry")
                task.wait(0.1)
                events.Carry:InvokeServer("Carry")
            end
        end)
    end

    -- إعداد Anti-AFK مع الوقت
    local function setupAntiAFK(state)
        AntiAFKEnabled = state
        if antiAfkTimer then antiAfkTimer:Destroy() antiAfkTimer = nil end
        if state then
            local timerGui = Instance.new("ScreenGui", CoreGui)
            timerGui.Name = "AFK_Timer_LAVA"
            local frame = Instance.new("Frame", timerGui)
            frame.Size = UDim2.new(0, 110, 0, 32)
            frame.Position = UDim2.new(0.9, -55, 0, 10)
            frame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
            frame.BackgroundTransparency = 0.2
            Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)
            local label = Instance.new("TextLabel", frame)
            label.Size = UDim2.new(1, 0, 1, 0)
            label.BackgroundTransparency = 1
            label.TextColor3 = Color3.fromRGB(0, 255, 100)
            label.Font = Enum.Font.GothamBold
            label.TextSize = 15
            label.Text = "05:00"
            
            local start = tick()
            antiAfkTimer = timerGui
            
            task.spawn(function()
                while AntiAFKEnabled do
                    local elapsed = tick() - start
                    local remaining = 300 - elapsed
                    if remaining <= 0 then
                        if player.Character and player.Character:FindFirstChild("Humanoid") then
                            player.Character.Humanoid.Jump = true
                        end
                        start = tick()
                        remaining = 300
                    end
                    local mins = math.floor(remaining / 60)
                    local secs = math.floor(remaining % 60)
                    label.Text = string.format("%02d:%02d", mins, secs)
                    task.wait(0.1)
                end
                timerGui:Destroy()
            end)
        end
    end

    -- FPS Boost
    local function toggleFPSBoost(state)
        FPSBoostEnabled = state
        if state then
            settings().Rendering.QualityLevel = 1
            Lighting.GlobalShadows = false
            Lighting.FogEnd = 100000
        else
            settings().Rendering.QualityLevel = 10
            Lighting.GlobalShadows = true
            Lighting.FogEnd = 10000
        end
    end

    -- Auto Buy Loops
    local function updateAutoBuyLoops()
        if autoBuy1Loop then autoBuy1Loop:Disconnect() end
        if autoBuy5Loop then autoBuy5Loop:Disconnect() end
        if autoBuy10Loop then autoBuy10Loop:Disconnect() end
        if autoBuyCarryLoop then autoBuyCarryLoop:Disconnect() end
        
        if AutoBuy1 then
            autoBuy1Loop = RunService.Heartbeat:Connect(function()
                if AutoBuy1 then buySpeed(1) task.wait(0.5) end
            end)
        end
        if AutoBuy5 then
            autoBuy5Loop = RunService.Heartbeat:Connect(function()
                if AutoBuy5 then buySpeed(5) task.wait(0.5) end
            end)
        end
        if AutoBuy10 then
            autoBuy10Loop = RunService.Heartbeat:Connect(function()
                if AutoBuy10 then buySpeed(10) task.wait(0.5) end
            end)
        end
        if AutoBuyCarry then
            autoBuyCarryLoop = RunService.Heartbeat:Connect(function()
                if AutoBuyCarry then buyCarryDouble() task.wait(0.5) end
            end)
        end
    end

    -- إنشاء واجهة LAVA
    local Container, TabBar = createUI("RXT ON TOP", Color3.fromRGB(255, 70, 70))

    -- التبويبات
    addTab(TabBar, "Main", function()
        clear(Container)
        
        local function createToggleWithFlag(text, flag, setter)
            createToggle(Container, text, flag, function(state)
                if setter then setter(state) end
            end)
        end

        createToggleWithFlag("🛡️ GOD Mode", isGodMode, function(v) 
            isGodMode = v
            if v then makeInvincible(v) end
        end)
        
        createToggleWithFlag("🦘 Infinite Jump", InfiniteJumpEnabled, function(v) InfiniteJumpEnabled = v end)
        createToggleWithFlag("🛡️ Anti-AFK", AntiAFKEnabled, function(v) setupAntiAFK(v) end)
        createToggleWithFlag("🔓 Unlock Zoom", UnlockZoomEnabled, function(v) 
            UnlockZoomEnabled = v
            player.CameraMaxZoomDistance = v and 5000 or 400
        end)
        createToggleWithFlag("⚡ FPS Boost", FPSBoostEnabled, function(v) toggleFPSBoost(v) end)
    end)

    addTab(TabBar, "Auto", function()
        clear(Container)
        
        local function createToggleWithFlag(text, flag, setter)
            createToggle(Container, text, flag, function(state)
                if setter then setter(state) end
                updateAutoBuyLoops()
            end)
        end

        createToggleWithFlag("💰 Speed 1", AutoBuy1, function(v) AutoBuy1 = v end)
        createToggleWithFlag("💰 Speed 5", AutoBuy5, function(v) AutoBuy5 = v end)
        createToggleWithFlag("💰 Speed 10", AutoBuy10, function(v) AutoBuy10 = v end)
        createToggleWithFlag("💪🏽 Carry", AutoBuyCarry, function(v) AutoBuyCarry = v end)
    end)

    addTab(TabBar, "⭐", function()
        clear(Container)
        
        local title = Instance.new("TextLabel", Container)
        title.Size = UDim2.new(1, 0, 0, 30)
        title.BackgroundTransparency = 1
        title.Text = "⭐ FEEDBACK"
        title.TextColor3 = Color3.fromRGB(255, 200, 0)
        title.Font = Enum.Font.GothamBold
        title.TextSize = 14
        
        local RatingBox = Instance.new("TextBox", Container)
        RatingBox.Size = UDim2.new(1, 0, 0, 40)
        RatingBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        RatingBox.PlaceholderText = "التقييم 0-10"
        RatingBox.Text = ""
        RatingBox.TextColor3 = Color3.new(1, 1, 1)
        RatingBox.Font = Enum.Font.Gotham
        RatingBox.TextSize = 13
        Instance.new("UICorner", RatingBox)

        local SuggestionBox = Instance.new("TextBox", Container)
        SuggestionBox.Size = UDim2.new(1, 0, 0, 70)
        SuggestionBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        SuggestionBox.PlaceholderText = "اقتراحك..."
        SuggestionBox.Text = ""
        SuggestionBox.TextColor3 = Color3.new(1, 1, 1)
        SuggestionBox.Font = Enum.Font.Gotham
        SuggestionBox.TextSize = 13
        SuggestionBox.TextWrapped = true
        SuggestionBox.MultiLine = true
        Instance.new("UICorner", SuggestionBox)

        local SendBtn = Instance.new("TextButton", Container)
        SendBtn.Size = UDim2.new(1, 0, 0, 40)
        SendBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
        SendBtn.Text = "إرسال"
        SendBtn.TextColor3 = Color3.new(1, 1, 1)
        SendBtn.Font = Enum.Font.GothamBold
        SendBtn.TextSize = 14
        Instance.new("UICorner", SendBtn)

        local StatusLabel = Instance.new("TextLabel", Container)
        StatusLabel.Size = UDim2.new(1, 0, 0, 25)
        StatusLabel.BackgroundTransparency = 1
        StatusLabel.Text = ""
        StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
        StatusLabel.Font = Enum.Font.Gotham
        StatusLabel.TextSize = 12

        SendBtn.MouseButton1Click:Connect(function()
            local rating = tonumber(RatingBox.Text)
            local suggestion = SuggestionBox.Text
            
            if not rating or rating < 0 or rating > 10 then
                StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
                StatusLabel.Text = "❌ التقييم من 0-10"
                return
            end
            
            if suggestion == "" then
                StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
                StatusLabel.Text = "❌ اكتب اقتراح"
                return
            end
            
            StatusLabel.TextColor3 = Color3.fromRGB(255, 200, 0)
            StatusLabel.Text = "⏱️ جاري الإرسال..."
            
            local sent = sendToDiscord(rating, suggestion, "LAVA")
            
            if sent then
                StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
                StatusLabel.Text = "✅ تم الإرسال!"
                RatingBox.Text = ""
                SuggestionBox.Text = ""
            else
                StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
                StatusLabel.Text = "❌ فشل الإرسال"
            end
        end)
    end)

    -- Infinite Jump
    UserInputService.JumpRequest:Connect(function()
        if InfiniteJumpEnabled and player.Character then
            player.Character.Humanoid:ChangeState("Jumping")
        end
    end)

    disableLavaTouch()
    print("✅ LAVA HUB شغال")
end

-- ============================================================================
-- سكريبت TSUNAMI
-- ============================================================================
local function runTsunamiScript()
    print("🌊 تم تشغيل TSUNAMI HUB")

    local InfiniteJumpEnabled = false
    local AntiAFKEnabled = false
    local UnlockZoomEnabled = false
    local AutoBuy1 = false
    local AutoBuy10 = false
    local IsMoving = false
    local FPSBoostEnabled = false
    local FarmMoneyEnabled = false

    local AutoCollect = { Enabled = false, Button = nil, Base = nil }
    local antiAfkTimer = nil
    local farmMoneyLoop = nil

    local STAGES = {
        [1] = Vector3.new(242, 7.5, 135.65),
        [2] = Vector3.new(341, 7.5, 135.65),
        [3] = Vector3.new(470, 7.5, 135.4),
        [4] = Vector3.new(649, 7.5, 135.4),
        [5] = Vector3.new(913, 7.5, 135.5),
        [6] = Vector3.new(1310, 7.5, 135.5),
        [7] = Vector3.new(1900, 7.5, 135.65),
        [8] = Vector3.new(2605, 7.5, 135.65),
        [9] = Vector3.new(3135.5, 7.5, 135.65),
        [10] = Vector3.new(3490.5, 7.5, 135.65),
        [11] = Vector3.new(3853.5, 7.5, 135.65)
    }

    local function GetRealStage()
        if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return 1 end
        local rootPos = player.Character.HumanoidRootPart.Position
        local closest, minDist = 1, math.huge
        for i, pos in pairs(STAGES) do
            local dist = (rootPos - pos).Magnitude
            if dist < minDist then minDist = dist closest = i end
        end
        return minDist > 30 and 0 or closest
    end

    local function GetNextStageForward()
        if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return 1 end
        local rootPos = player.Character.HumanoidRootPart.Position
        local forwardStage, minDist = 11, math.huge
        for i, pos in pairs(STAGES) do
            if pos.X > rootPos.X then
                local dist = (rootPos - pos).Magnitude
                if dist < minDist then minDist = dist forwardStage = i end
            end
        end
        return forwardStage
    end

    local function GetNextStageBackward()
        if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return 11 end
        local rootPos = player.Character.HumanoidRootPart.Position
        local backwardStage, minDist = 1, math.huge
        for i, pos in pairs(STAGES) do
            if pos.X < rootPos.X then
                local dist = (rootPos - pos).Magnitude
                if dist < minDist then minDist = dist backwardStage = i end
            end
        end
        return backwardStage
    end

    local function TeleportToStage(targetStage)
        if IsMoving or not STAGES[targetStage] or not player.Character then return end
        IsMoving = true
        local char = player.Character
        local root = char.HumanoidRootPart
        local noclip = RunService.Stepped:Connect(function()
            for _, v in pairs(char:GetDescendants()) do
                if v:IsA("BasePart") then v.CanCollide = false end
            end
        end)
        local tween = TweenService:Create(root, TweenInfo.new((root.Position - STAGES[targetStage]).Magnitude / 850, Enum.EasingStyle.Linear), {CFrame = CFrame.new(STAGES[targetStage])})
        tween:Play()
        tween.Completed:Connect(function()
            noclip:Disconnect()
            for _, v in pairs(char:GetDescendants()) do
                if v:IsA("BasePart") then v.CanCollide = true end
            end
            IsMoving = false
        end)
    end

    local function MoveNext()
        local cur = GetRealStage()
        if cur == 0 then TeleportToStage(GetNextStageForward())
        else TeleportToStage(math.min(cur + 1, 11)) end
    end

    local function MovePrev()
        local cur = GetRealStage()
        if cur == 0 then TeleportToStage(GetNextStageBackward())
        else TeleportToStage(math.max(cur - 1, 1)) end
    end

    local function ToggleFPSBoost(state)
        FPSBoostEnabled = state
        if state then
            settings().Rendering.QualityLevel = 1
            Lighting.GlobalShadows = false
            Lighting.FogEnd = 100000
        else
            settings().Rendering.QualityLevel = 10
            Lighting.GlobalShadows = true
            Lighting.FogEnd = 10000
        end
    end

    local function setupAutoCollect()
        local basesFolder = workspace:FindFirstChild("Bases")
        local function getNearestBase()
            local char = player.Character
            if not char or not char:FindFirstChild("HumanoidRootPart") or not basesFolder then return nil end
            local myPos, closest, dist = char.HumanoidRootPart.Position, nil, math.huge
            for _, base in pairs(basesFolder:GetChildren()) do
                local pos = base:IsA("Model") and base:GetPivot().Position or base.Position
                if (myPos - pos).Magnitude < dist then dist, closest = (myPos - pos).Magnitude, base end
            end
            return closest
        end
        local function collectEverything()
            if not AutoCollect.Base or not AutoCollect.Enabled then return end
            local char = player.Character
            if not char or not char:FindFirstChild("HumanoidRootPart") then return end
            for _, item in pairs(AutoCollect.Base:GetDescendants()) do
                if not AutoCollect.Enabled then break end
                if item.Name == "Collect" then
                    local targetPos = item:IsA("Model") and item:GetPivot().Position or (item:IsA("BasePart") and item.Position)
                    if targetPos then
                        char.HumanoidRootPart.CFrame = CFrame.new(targetPos)
                        task.wait(0.4)
                    end
                end
            end
        end
        AutoCollect.Enabled = false
        return { toggle = function()
            AutoCollect.Enabled = not AutoCollect.Enabled
            if AutoCollect.Enabled then
                if AutoCollect.Button then AutoCollect.Button.Text, AutoCollect.Button.BackgroundColor3 = "Auto Collect: ON", Color3.fromRGB(0, 180, 0) end
                if player.Character then player.Character:BreakJoints() end
                player.CharacterAdded:Wait(); task.wait(3)
                AutoCollect.Base = getNearestBase()
                if AutoCollect.Base then task.spawn(function() while AutoCollect.Enabled do collectEverything(); task.wait(2) end end)
                else AutoCollect.Enabled = false
                    if AutoCollect.Button then AutoCollect.Button.Text, AutoCollect.Button.BackgroundColor3 = "Auto Collect: ERROR", Color3.fromRGB(180, 0, 0) end
                end
            else if AutoCollect.Button then AutoCollect.Button.Text, AutoCollect.Button.BackgroundColor3 = "Auto Collect: OFF", Color3.fromRGB(35, 36, 45) end end
        end }
    end

    -- Auto Farm Money
    local FarmMoneyPosition = Vector3.new(423.992, -15.5, -338)

    local function ToggleFarmMoney(state)
        FarmMoneyEnabled = state
        if farmMoneyLoop then farmMoneyLoop:Disconnect(); farmMoneyLoop = nil end
        if state then
            farmMoneyLoop = RunService.Heartbeat:Connect(function()
                if not FarmMoneyEnabled then return end
                local char = player.Character
                if not char or not char:FindFirstChild("HumanoidRootPart") then return end
                local root = char.HumanoidRootPart
                if (root.Position - FarmMoneyPosition).Magnitude > 5 then
                    for _, v in pairs(char:GetDescendants()) do
                        if v:IsA("BasePart") then v.CanCollide = false end
                    end
                    root.CFrame = CFrame.new(FarmMoneyPosition)
                    root.Velocity = Vector3.zero
                end
            end)
        end
    end

    -- إعداد Anti-AFK مع الوقت
    local function setupAntiAFK(state)
        AntiAFKEnabled = state
        if antiAfkTimer then antiAfkTimer:Destroy() antiAfkTimer = nil end
        if state then
            local timerGui = Instance.new("ScreenGui", CoreGui)
            timerGui.Name = "AFK_Timer_TSUNAMI"
            local frame = Instance.new("Frame", timerGui)
            frame.Size = UDim2.new(0, 110, 0, 32)
            frame.Position = UDim2.new(0.9, -55, 0, 10)
            frame.BackgroundColor3 = Color3.new(0, 0, 0)
            frame.BackgroundTransparency = 0.2
            Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)
            local label = Instance.new("TextLabel", frame)
            label.Size = UDim2.new(1, 0, 1, 0)
            label.BackgroundTransparency = 1
            label.TextColor3 = Color3.new(0, 255, 100)
            label.Font = Enum.Font.GothamBold
            label.TextSize = 15
            label.Text = "05:00"
            
            local start = tick()
            antiAfkTimer = timerGui
            
            task.spawn(function()
                while AntiAFKEnabled do
                    local elapsed = tick() - start
                    local remaining = 300 - elapsed
                    if remaining <= 0 then
                        if player.Character and player.Character:FindFirstChild("Humanoid") then
                            player.Character.Humanoid.Jump = true
                        end
                        start = tick()
                        remaining = 300
                    end
                    local mins = math.floor(remaining / 60)
                    local secs = math.floor(remaining % 60)
                    label.Text = string.format("%02d:%02d", mins, secs)
                    task.wait(0.1)
                end
                timerGui:Destroy()
            end)
        end
    end

    -- إنشاء واجهة TSUNAMI
    local Container, TabBar, ScreenGui = createUI("RXT ON TOP", Color3.fromRGB(45, 130, 255))
    local autoCollectSystem = setupAutoCollect()

    -- التبويبات
    addTab(TabBar, "Main", function()
        clear(Container)
        
        local function createToggleWithFlag(text, flag, setter)
            createToggle(Container, text, flag, function(state)
                if setter then setter(state) end
            end)
        end

        createToggleWithFlag("🦘 Infinite Jump", InfiniteJumpEnabled, function(v) InfiniteJumpEnabled = v end)
        createToggleWithFlag("🛡️ Anti-AFK", AntiAFKEnabled, function(v) setupAntiAFK(v) end)
        createToggleWithFlag("🔓 Unlock Zoom", UnlockZoomEnabled, function(v) 
            UnlockZoomEnabled = v
            player.CameraMaxZoomDistance = v and 5000 or 400
        end)
        createToggleWithFlag("⚡ FPS Boost", FPSBoostEnabled, function(v) ToggleFPSBoost(v) end)
    end)

    addTab(TabBar, "Auto", function()
        clear(Container)
        
        local function createToggleWithFlag(text, flag, setter)
            createToggle(Container, text, flag, function(state)
                if setter then setter(state) end
            end)
        end

        createToggleWithFlag("💰 Speed 1", AutoBuy1, function(v) AutoBuy1 = v end)
        createToggleWithFlag("💰 Speed 10", AutoBuy10, function(v) AutoBuy10 = v end)

        local autoCollectBtn = Instance.new("TextButton", Container)
        autoCollectBtn.Size = UDim2.new(1, 0, 0, 40)
        autoCollectBtn.BackgroundColor3 = Color3.fromRGB(35, 37, 48)
        autoCollectBtn.Text = "📦 Auto Collect: OFF"
        autoCollectBtn.TextColor3 = Color3.new(1,1,1)
        autoCollectBtn.Font = Enum.Font.GothamBold
        autoCollectBtn.TextSize = 12
        Instance.new("UICorner", autoCollectBtn)
        AutoCollect.Button = autoCollectBtn
        autoCollectBtn.MouseButton1Click:Connect(function() autoCollectSystem.toggle() end)
    end)

    addTab(TabBar, "Event", function()
        clear(Container)
        
        local function createToggleWithFlag(text, flag, setter)
            createToggle(Container, text, flag, function(state)
                if setter then setter(state) end
            end)
        end

        createToggleWithFlag("💰 Auto Farm Money", FarmMoneyEnabled, function(v) ToggleFarmMoney(v) end)

        -- زر VIP GUI
        local vipBtn = Instance.new("TextButton", Container)
        vipBtn.Size = UDim2.new(1, 0, 0, 40)
        vipBtn.BackgroundColor3 = Color3.fromRGB(255, 215, 0)
        vipBtn.Text = "👑 VIP GUI"
        vipBtn.TextColor3 = Color3.new(0, 0, 0)
        vipBtn.Font = Enum.Font.GothamBold
        vipBtn.TextSize = 13
        Instance.new("UICorner", vipBtn)
        
        vipBtn.MouseButton1Click:Connect(function()
            -- نافذة VIP المنفصلة
            local vFrame = Instance.new("Frame", ScreenGui)
            vFrame.Size = UDim2.new(0, 200, 0, 240)
            vFrame.Position = UDim2.new(0.7, 0, 0.4, 0)
            vFrame.BackgroundColor3 = Color3.fromRGB(25, 26, 32)
            vFrame.Active = true
            vFrame.Draggable = true
            Instance.new("UICorner", vFrame).CornerRadius = UDim.new(0, 8)
            
            -- عنوان النافذة
            local vt = Instance.new("TextLabel", vFrame)
            vt.Size = UDim2.new(1, 0, 0, 35)
            vt.Text = "VIP CONTROLS"
            vt.BackgroundColor3 = Color3.fromRGB(45, 130, 255)
            vt.TextColor3 = Color3.new(1, 1, 1)
            vt.Font = Enum.Font.GothamBold
            vt.TextSize = 14
            Instance.new("UICorner", vt).CornerRadius = UDim.new(0, 8)
            
            -- عرض المرحلة الحالية
            local st = Instance.new("TextLabel", vFrame)
            st.Size = UDim2.new(1, 0, 0, 25)
            st.Position = UDim2.new(0, 0, 0, 40)
            st.BackgroundTransparency = 1
            st.TextColor3 = Color3.new(1, 1, 0)
            st.Font = Enum.Font.GothamBold
            st.TextSize = 12
            task.spawn(function() 
                while vFrame and vFrame.Parent do 
                    local stage = GetRealStage()
                    st.Text = "STAGE: " .. (stage == 0 and "OUT" or tostring(stage)) 
                    task.wait(0.3) 
                end 
            end)
            
            -- دالة إنشاء الأزرار
            local function sBtn(txt, y, func)
                local sb = Instance.new("TextButton", vFrame)
                sb.Size = UDim2.new(0.9, 0, 0, 35)
                sb.Position = UDim2.new(0.05, 0, 0, y)
                sb.Text = txt
                sb.BackgroundColor3 = Color3.fromRGB(45, 46, 55)
                sb.TextColor3 = Color3.new(1, 1, 1)
                sb.Font = Enum.Font.GothamBold
                sb.TextSize = 12
                Instance.new("UICorner", sb).CornerRadius = UDim.new(0, 5)
                sb.MouseButton1Click:Connect(func)
            end
            
            -- أزرار التنقل
            sBtn("⬆️ NEXT STAGE", 75, MoveNext)
            sBtn("⬇️ PREV STAGE", 115, MovePrev)
            sBtn("🔓 UNLOCK DOORS", 155, function()
                for _, v in pairs(workspace:GetDescendants()) do
                    if v.Name == "VIPWalls" then
                        for _, c in pairs(v:GetDescendants()) do
                            if c:IsA("BasePart") then c.CanCollide, c.Transparency = false, 0.5 end
                        end
                    end
                end
            end)
            
            -- زر إغلاق النافذة
            local cV = Instance.new("TextButton", vFrame)
            cV.Size = UDim2.new(0.9, 0, 0, 30)
            cV.Position = UDim2.new(0.05, 0, 0, 200)
            cV.Text = "CLOSE"
            cV.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
            cV.TextColor3 = Color3.new(1, 1, 1)
            cV.Font = Enum.Font.GothamBold
            cV.TextSize = 12
            Instance.new("UICorner", cV).CornerRadius = UDim.new(0, 5)
            cV.MouseButton1Click:Connect(function() vFrame:Destroy() end)
        end)
    end)

    addTab(TabBar, "⭐", function()
        clear(Container)
        
        local title = Instance.new("TextLabel", Container)
        title.Size = UDim2.new(1, 0, 0, 30)
        title.BackgroundTransparency = 1
        title.Text = "⭐ FEEDBACK"
        title.TextColor3 = Color3.fromRGB(255, 200, 0)
        title.Font = Enum.Font.GothamBold
        title.TextSize = 14
        
        local RatingBox = Instance.new("TextBox", Container)
        RatingBox.Size = UDim2.new(1, 0, 0, 40)
        RatingBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        RatingBox.PlaceholderText = "التقييم 0-10"
        RatingBox.Text = ""
        RatingBox.TextColor3 = Color3.new(1, 1, 1)
        RatingBox.Font = Enum.Font.Gotham
        RatingBox.TextSize = 13
        Instance.new("UICorner", RatingBox)

        local SuggestionBox = Instance.new("TextBox", Container)
        SuggestionBox.Size = UDim2.new(1, 0, 0, 70)
        SuggestionBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        SuggestionBox.PlaceholderText = "اقتراحك..."
        SuggestionBox.Text = ""
        SuggestionBox.TextColor3 = Color3.new(1, 1, 1)
        SuggestionBox.Font = Enum.Font.Gotham
        SuggestionBox.TextSize = 13
        SuggestionBox.TextWrapped = true
        SuggestionBox.MultiLine = true
        Instance.new("UICorner", SuggestionBox)

        local SendBtn = Instance.new("TextButton", Container)
        SendBtn.Size = UDim2.new(1, 0, 0, 40)
        SendBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
        SendBtn.Text = "إرسال"
        SendBtn.TextColor3 = Color3.new(1, 1, 1)
        SendBtn.Font = Enum.Font.GothamBold
        SendBtn.TextSize = 14
        Instance.new("UICorner", SendBtn)

        local StatusLabel = Instance.new("TextLabel", Container)
        StatusLabel.Size = UDim2.new(1, 0, 0, 25)
        StatusLabel.BackgroundTransparency = 1
        StatusLabel.Text = ""
        StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
        StatusLabel.Font = Enum.Font.Gotham
        StatusLabel.TextSize = 12

        SendBtn.MouseButton1Click:Connect(function()
            local rating = tonumber(RatingBox.Text)
            local suggestion = SuggestionBox.Text
            
            if not rating or rating < 0 or rating > 10 then
                StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
                StatusLabel.Text = "❌ التقييم من 0-10"
                return
            end
            
            if suggestion == "" then
                StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
                StatusLabel.Text = "❌ اكتب اقتراح"
                return
            end
            
            StatusLabel.TextColor3 = Color3.fromRGB(255, 200, 0)
            StatusLabel.Text = "⏱️ جاري الإرسال..."
            
            local sent = sendToDiscord(rating, suggestion, "TSUNAMI")
            
            if sent then
                StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
                StatusLabel.Text = "✅ تم الإرسال!"
                RatingBox.Text = ""
                SuggestionBox.Text = ""
            else
                StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
                StatusLabel.Text = "❌ فشل الإرسال"
            end
        end)
    end)

    -- حلقة التشغيل المستمر
    task.spawn(function()
        while task.wait(0.3) do
            pcall(function()
                if AutoBuy1 then ReplicatedStorage.RemoteFunctions.UpgradeSpeed:InvokeServer(1) end
                if AutoBuy10 then ReplicatedStorage.RemoteFunctions.UpgradeSpeed:InvokeServer(10) end
            end)
        end
    end)

    UserInputService.JumpRequest:Connect(function()
        if InfiniteJumpEnabled and player.Character then
            player.Character.Humanoid:ChangeState("Jumping")
        end
    end)

    print("✅ TSUNAMI HUB شغال")
end

-- ============================================================================
-- MAIN - التشغيل التلقائي
-- ============================================================================

wait(2)
local mapName, mapDisplay = showMapName()
wait(1)
local gameMode = detectGameMode(mapName)
print("🎮 وضع اللعبة: " .. gameMode)

-- إرسال إشعار التشغيل
sendRunNotification(mapName)

if gameMode == "LAVA" then
    mapDisplay:Destroy()
    runLavaScript()
elseif gameMode == "TSUNAMI" then
    mapDisplay:Destroy()
    runTsunamiScript()
else
    -- تغيير لون العرض
    local frame = mapDisplay:FindFirstChildWhichIsA("Frame")
    if frame then
        local accent = frame:FindFirstChild("AccentLine")
        if accent then accent.BackgroundColor3 = Color3.fromRGB(255, 200, 0) end
        local text = frame:FindFirstChildWhichIsA("TextLabel")
        if text then text.Text = "⚠️ اختر يدوياً" end
    end

    -- قائمة اختيار يدوي
    local selectorGui = Instance.new("ScreenGui", CoreGui)
    selectorGui.Name = "ManualSelector"
    selectorGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    local frame = Instance.new("Frame", selectorGui)
    frame.Size = UDim2.new(0, 315, 0, 150)
    frame.Position = UDim2.new(0.5, -157, 0.5, -75)
    frame.BackgroundColor3 = Color3.fromRGB(20, 22, 28)
    frame.Active = true
    frame.Draggable = true
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 12)

    local title = Instance.new("TextLabel", frame)
    title.Size = UDim2.new(1, 0, 0, 40)
    title.BackgroundColor3 = Color3.fromRGB(35, 36, 45)
    title.Text = "🎮 اختر وضع اللعبة"
    title.TextColor3 = Color3.new(1, 1, 1)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 15
    Instance.new("UICorner", title).CornerRadius = UDim.new(0, 12)

    local lavaBtn = Instance.new("TextButton", frame)
    lavaBtn.Size = UDim2.new(0.4, 0, 0, 40)
    lavaBtn.Position = UDim2.new(0.05, 0, 0.6, 0)
    lavaBtn.BackgroundColor3 = Color3.fromRGB(255, 80, 80)
    lavaBtn.Text = "🔥 LAVA"
    lavaBtn.TextColor3 = Color3.new(1, 1, 1)
    lavaBtn.Font = Enum.Font.GothamBold
    lavaBtn.TextSize = 14
    Instance.new("UICorner", lavaBtn).CornerRadius = UDim.new(0, 8)

    local tsunamiBtn = Instance.new("TextButton", frame)
    tsunamiBtn.Size = UDim2.new(0.4, 0, 0, 40)
    tsunamiBtn.Position = UDim2.new(0.55, 0, 0.6, 0)
    tsunamiBtn.BackgroundColor3 = Color3.fromRGB(45, 130, 255)
    tsunamiBtn.Text = "🌊 TSUNAMI"
    tsunamiBtn.TextColor3 = Color3.new(1, 1, 1)
    tsunamiBtn.Font = Enum.Font.GothamBold
    tsunamiBtn.TextSize = 14
    Instance.new("UICorner", tsunamiBtn).CornerRadius = UDim.new(0, 8)

    lavaBtn.MouseButton1Click:Connect(function()
        selectorGui:Destroy()
        mapDisplay:Destroy()
        runLavaScript()
    end)

    tsunamiBtn.MouseButton1Click:Connect(function()
        selectorGui:Destroy()
        mapDisplay:Destroy()
        runTsunamiScript()
    end)
end

print("✅ BRAINROTS ULTIMATE SYSTEM يعمل")
