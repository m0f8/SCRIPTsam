local Players = game:GetService("Players")
local localPlayer = Players.LocalPlayer

local currentVehicleSeat = nil
local activeConstraints = {}

-- دالة لتنظيف الـ Constraints وإرجاع التصادم طبيعي
local function clearNoclipConstraints()
    for _, constraint in ipairs(activeConstraints) do
        if constraint and constraint.Parent then
            constraint:Destroy()
        end
    end
    activeConstraints = {}
end

local function startVehicleNoclip(character, vehicle)
    clearNoclipConstraints()
    
    local charParts = {}
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            table.insert(charParts, part)
        end
    end
    
    local vehParts = {}
    for _, part in ipairs(vehicle:GetDescendants()) do
        if part:IsA("BasePart") then
            table.insert(vehParts, part)
        end
    end
    
    -- عمل NoCollision بين اللاعب وكل أجزاء السيارة والكفرات
    for _, charPart in ipairs(charParts) do
        for _, vehPart in ipairs(vehParts) do
            local noCollide = Instance.new("NoCollisionConstraint")
            noCollide.Part0 = charPart
            noCollide.Part1 = vehPart
            noCollide.Parent = charPart
            table.insert(activeConstraints, noCollide)
        end
    end
    
    -- جعل وقت النيوكليب ثانيتين تماماً كما طلبت
    task.delay(2, function()
        clearNoclipConstraints()
    end)
end

local function setupCharacter(character)
    local humanoid = character:WaitForChild("Humanoid", 10)
    local rootPart = character:WaitForChild("HumanoidRootPart", 10)
    if not humanoid or not rootPart then return end
    
    -- مراقبة الكراسي (أمامية وخلفية)
    humanoid:GetPropertyChangedSignal("SeatPart"):Connect(function()
        if humanoid.SeatPart and (humanoid.SeatPart:IsA("VehicleSeat") or humanoid.SeatPart:IsA("Seat")) then
            currentVehicleSeat = humanoid.SeatPart
        end
    end)

    humanoid:GetPropertyChangedSignal("Sit"):Connect(function()
        if not humanoid.Sit and currentVehicleSeat then
            local seatRef = currentVehicleSeat
            currentVehicleSeat = nil
            
            local vehicle = seatRef:FindFirstAncestorOfClass("Model") or seatRef.Parent
            
            if vehicle then
                -- 1. تشغيل منع التصادم الشامل (مع الكفرات والهيكل)
                startVehicleNoclip(character, vehicle)
                
                -- 2. تصفير السرعة لمنع أي اندفاع
                rootPart.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                rootPart.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
                
                -- 3. إرجاعك ورسبنتك في وسط المقعد تماماً (بدون إخراجك للخارج)
                task.defer(function()
                    if rootPart and seatRef then
                        -- وضعتك في نفس إحداثيات المقعد تماماً (مع رفع بسيط جداً 0.5 لتجنب الالتصاق بالأرضية)
                        rootPart.CFrame = seatRef.CFrame + Vector3.new(0, 0.5, 0)
                        rootPart.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                    end
                end)
            end
        end
    end)
end

if localPlayer.Character then
    setupCharacter(localPlayer.Character)
end

localPlayer.CharacterAdded:Connect(setupCharacter)
