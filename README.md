-- // Configurations \\ --
_G.Configurations = {
   -- KeyBinds --
   ToggleKeyBind = Enum.KeyCode.H,
   PredictionKeyBind = Enum.KeyCode.J;

   -- Settings --
   FOV = 1250;

   -- Filters --
   Filters = {
       "Player1"
   };
   FilterType = "Whitelist";

   -- Aimbot sensitivity --
   Gyro = {
       Power = 15000,                  -- Higher the number, the faster you will spin.
       Dampening = 0,                  -- The lower it is, the more it will look "snappy"
   };
}
-- // Constants \\ --
-- [ Services ] --
local Services = setmetatable({}, {__index = function(Self, Index)
local NewService = game:GetService(Index)
if NewService then
Self[Index] = NewService
end
return NewService
end})



-- [ Modules ] --
local Drawing = loadstring(game:HttpGet("https://raw.githubusercontent.com/iHavoc101/Genesis-Studios/main/Modules/DrawingAPI.lua", true))()

-- [ Game Settings ] --
local GameSettings = UserSettings():GetService("UserGameSettings")

-- [ LocalPlayer ] --
local LocalPlayer = Services.Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- [ Interface ] --
local FOVCircle = Drawing.new("Circle", {
   Thickness = 2.5,
   Color = Color3.fromRGB(150, 150, 150),
   NumSides = 20,
   Radius = _G.Configurations.FOV
})

local Target = Drawing.new("Triangle", {
   Thickness = 5,
   Color = Color3.fromRGB(0, 200, 255)
})

-- // Variables \\ --
local SilentAimToggle = true
local PredictionMultiplier = 2.5
getgenv().Key = nil

local ClosestTarget = nil

-- [ Body Gyro ] --
local BodyGyro = Instance.new("BodyGyro")
BodyGyro.P = _G.Configurations.Gyro.Power
BodyGyro.D = _G.Configurations.Gyro.Dampening
BodyGyro.MaxTorque = Vector3.new(0, math.huge, 0)

-- [ Raycast Parameters ] --
local RaycastParameters = RaycastParams.new()
RaycastParameters.IgnoreWater = true
RaycastParameters.FilterType = Enum.RaycastFilterType.Blacklist
RaycastParameters.FilterDescendantsInstances = {LocalPlayer.Character}

-- // Functions \\ --
local function NotObstructing(Destination, Ancestor)
   -- [ Camera ] --
   local ObstructingParts = Camera:GetPartsObscuringTarget({Destination}, {Ancestor, LocalPlayer.Character})

   for i,v in ipairs(ObstructingParts) do
       pcall(function()
           if v.Transparency >= 1 then
               table.remove(ObstructingParts, i)
           end
       end)
   end

   if #ObstructingParts <= 0 then
       return true
   end

   -- [ Raycast ] --
   RaycastParameters.FilterDescendantsInstances = {LocalPlayer.Character}

   local Origin = Camera.CFrame.Position
   local Direction = (Destination - Origin).Unit * 500
   local RayResult = workspace:Raycast(Origin, Direction, RaycastParameters) or {
       Instance = nil;
       Position = Origin + Direction;
       Material = Enum.Material.Air;
   }

   if RayResult.Instance and (RayResult.Instance:IsDescendantOf(Ancestor) or RayResult.Instance == Ancestor) then
       return true
   end

   -- [ Obstructed ] --
   return false
end

local function ValidCharacter(Character)
   return Character and (Character:FindFirstChildWhichIsA("Humanoid") and Character:FindFirstChildWhichIsA("Humanoid").Health ~= 0) or false
end

local function GetTarget(Distance, IgnoreList)
   local MousePosition = Services.UserInputService:GetMouseLocation()

   local Closest = nil
   local Position = Vector2.new(0, 0)
   local ShortestDistance = Distance or math.huge

   for i,v in ipairs(Services.Players:GetPlayers()) do
       if _G.Configurations.FilterType == "Whitelist" and table.find(_G.Configurations.Filters, v.Name) then
           continue
       elseif _G.Configurations.FilterType == "Blacklist" and table.find(_G.Configurations.Filters, v.Name) == nil then
           continue
       end

       if v ~= LocalPlayer and ValidCharacter(v.Character) then
           local ViewportPosition, OnScreen = Camera:WorldToViewportPoint(v.Character.PrimaryPart.Position)
           local Magnitude = (Vector2.new(ViewportPosition.X, ViewportPosition.Y) - MousePosition).Magnitude

           if OnScreen == false then
               continue
           end

           if Magnitude < ShortestDistance and NotObstructing(v.Character.PrimaryPart.Position, v.Character) == true then
               Closest = v
               Position = ViewportPosition
               ShortestDistance = Magnitude
           end
       end
   end

   return Closest, Position, ShortestDistance
end

-- // Events \\ --
Services.RunService.Heartbeat:Connect(function()

   -- Aim --
   local Closest, Position, Distance = GetTarget(_G.Configurations.FOV)
   local MouseLocation = Services.UserInputService:GetMouseLocation()

   ClosestTarget = Closest
   if Closest then
       BodyGyro.Parent = LocalPlayer.Character.PrimaryPart

       local Velocity = Closest.Character.PrimaryPart:GetVelocityAtPosition(Closest.Character.PrimaryPart.Position) * Vector3.new(1, 0, 1)
       local Magnitude = (Closest.Character.PrimaryPart.Position - LocalPlayer.Character.PrimaryPart.Position).Magnitude
       local Prediction = Velocity * (PredictionMultiplier / 10) * Magnitude / 100
       if Velocity.Magnitude == 0 then
           Prediction = Vector3.new(0, 0, 0)
       end

       if SilentAimToggle == true and Prediction.Magnitude >= 0 then
           BodyGyro.CFrame = CFrame.new(LocalPlayer.Character.PrimaryPart.Position, Closest.Character.PrimaryPart.Position + Prediction)
       else
           BodyGyro.Parent = nil
       end

       Target.PointA = Vector2.new(Position.X - 25, Position.Y + 25)
       Target.PointB = Vector2.new(Position.X + 25, Position.Y + 25)
       Target.PointC = Vector2.new(Position.X, Position.Y - 25)
   else
       BodyGyro.Parent = nil

       Target.PointA = Vector2.new(MouseLocation.X - 25, MouseLocation.Y + 25)
       Target.PointB = Vector2.new(MouseLocation.X + 25, MouseLocation.Y + 25)
       Target.PointC = Vector2.new(MouseLocation.X, MouseLocation.Y - 25)
   end

   -- Update --
   BodyGyro.P = _G.Configurations.Gyro.Power
   BodyGyro.D = _G.Configurations.Gyro.Dampening

   -- Drawing --
   FOVCircle.Position = MouseLocation
end)

-- // Binds \\ --
Services.RunService:BindToRenderStep("SetCameraRotation", Enum.RenderPriority.Camera.Value + 1, function()
   if SilentAimToggle == true and ClosestTarget ~= nil then
       GameSettings.RotationType = Enum.RotationType.MovementRelative
   end
end)

Services.ContextActionService:BindAction("ToggleSilent", function(actionName, InputState, inputObject)
   if InputState == Enum.UserInputState.End then
return
   end
   SilentAimToggle = not SilentAimToggle
   Services.StarterGui:SetCore("SendNotification", {
       Title = "Silent Aim";
Text = "Enabled: " .. tostring(SilentAimToggle);
Icon = SilentAimToggle and "rbxassetid://6031068430" or "rbxassetid://6031068429";
Duration = 5;
Button1 = "Dismiss"
   })
end, false, _G.Configurations.ToggleKeyBind)

Services.ContextActionService:BindAction("TogglePrediction", function(actionName, InputState, inputObject)
   if InputState == Enum.UserInputState.End then
return
   end
   PredictionMultiplier = PredictionMultiplier == 2.5 and 10 or 2.5
   Services.StarterGui:SetCore("SendNotification", {
       Title = "Prediction";
Text = "Enabled: " .. tostring(PredictionMultiplier == 10);
Icon = "rbxassetid://6034287516";
Duration = 5;
Button1 = "Dismiss"
   })
end, false, _G.Configurations.PredictionKeyBind)

-- // Bypass \\ --
for _,v in ipairs(getconnections(Services.ScriptContext.Error)) do
   pcall(function()
       v:Disable()
   end)
end

return {}
