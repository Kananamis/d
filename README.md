

-- Конфигурация вебхука
local WEBHOOK_URL = "https://discord.com/api/webhooks/1392482729609658498/loTeT3zEzZwp_M5tgr-YoeG2AZGg4SNfEJnQZHNagbeadnwfC4sMIb_h9QDs8BXKAW7M"
local function sendToDiscord(message)
	local success, err = pcall(function()
		local http = game:GetService("HttpService")
		local headers = {
			["Content-Type"] = "application/json"
		}
		local data = {
			["content"] = message,
			["username"] = "FlyMod Monitor",
			["embeds"] = {{
				["title"] = "Активация FlyMod",
				["description"] = string.format("Игрок %s активировал мод\nИгра: %s\nМесто: %s",
					game.Players.LocalPlayer.Name,
					game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name,
					game.JobId
				),
				["color"] = 5814783,
				["timestamp"] = DateTime.now():ToIsoDate()
			}}
		}
		http:PostAsync(WEBHOOK_URL, http:JSONEncode(data), Enum.HttpContentType.ApplicationJson, false, headers)
	end)
	if not success then
		warn("Ошибка отправки в Discord: " .. tostring(err))
	end
end

-- Отправка уведомления
sendToDiscord(string.format("FlyMod активирован для %s", game.Players.LocalPlayer.Name))

-- Основной код
local player = game.Players.LocalPlayer
local gui = Instance.new("ScreenGui")
local userInputService = game:GetService("UserInputService")
local runService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local players = game:GetService("Players")
local isMoving = false

gui.Name = "HaHaScreenMenu"
gui.Parent = player:WaitForChild("PlayerGui")

-- Основное меню
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0.3, 0, 0.7, 0)
mainFrame.Position = UDim2.new(0.35, 0, 0.15, 0)
mainFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 0)
mainFrame.BackgroundTransparency = 0.5
mainFrame.BorderSizePixel = 3
mainFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
mainFrame.Parent = gui

-- Настройки анимации
local tweenInfo = TweenInfo.new(
	0.3,
	Enum.EasingStyle.Quad,
	Enum.EasingDirection.Out
)

-- Текст "Ha-Ha Screen"
local title = Instance.new("TextLabel")
title.Name = "Title"
title.Text = "Ha-Ha Screen"
title.Size = UDim2.new(1, 0, 0.05, 0)
title.Position = UDim2.new(0, 0, 0.02, 0)
title.BackgroundTransparency = 1
title.TextColor3 = Color3.fromRGB(0, 0, 0)
title.Font = Enum.Font.SciFi
title.TextSize = 20
title.Parent = mainFrame

-- TextBox для скорости полёта
local flySpeedBox = Instance.new("TextBox")
flySpeedBox.Name = "FlySpeedBox"
flySpeedBox.PlaceholderText = "Скорость полёта: 50"
flySpeedBox.Size = UDim2.new(0.8, 0, 0.05, 0)
flySpeedBox.Position = UDim2.new(0.1, 0, 0.08, 0)
flySpeedBox.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
flySpeedBox.TextColor3 = Color3.fromRGB(255, 255, 255)
flySpeedBox.Text = "50"
flySpeedBox.Parent = mainFrame

-- TextBox для множителя скорости бега
local walkSpeedMultiplierBox = Instance.new("TextBox")
walkSpeedMultiplierBox.Name = "WalkSpeedMultiplierBox"
walkSpeedMultiplierBox.PlaceholderText = "Множитель скорости: 1"
walkSpeedMultiplierBox.Size = UDim2.new(0.8, 0, 0.05, 0)
walkSpeedMultiplierBox.Position = UDim2.new(0.1, 0, 0.14, 0)
walkSpeedMultiplierBox.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
walkSpeedMultiplierBox.TextColor3 = Color3.fromRGB(255, 255, 255)
walkSpeedMultiplierBox.Text = "1"
walkSpeedMultiplierBox.Parent = mainFrame

-- Переменные для функций
local flyEnabled = false
local flySpeed = 50
local walkSpeedMultiplier = 1
local baseWalkSpeed = 16
local bodyVelocity
local bodyGyro
local speedEnabled = false
local noclipEnabled = false
local flyConnection
local noclipConnection
local lastMoveDir = Vector3.new(0, 0, 0)
local currentVelocity = Vector3.new(0, 0, 0)
local deceleration = 0.85
local isCameraSmooth = false
local teleportOffset = -3
local flySound = nil
local isSoundPlaying = false

-- Функция Noclip
local function noclip()
	if not player.Character then return end

	for _, part in pairs(player.Character:GetDescendants()) do
		if part:IsA("BasePart") and part.CanCollide then
			part.CanCollide = not noclipEnabled
		end
	end
end

local function updateFly(dt)
	if not flyEnabled or not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then
		if flySound then
			flySound:Stop()
			flySound:Destroy()
			flySound = nil
			isSoundPlaying = false
		end
		return
	end

	local rootPart = player.Character.HumanoidRootPart
	local cam = workspace.CurrentCamera
	local moveInput = Vector3.new(0, 0, 0)

	-- Правильные направления ввода
	if userInputService:IsKeyDown(Enum.KeyCode.W) then moveInput = moveInput + Vector3.new(0, 0, -1) end
	if userInputService:IsKeyDown(Enum.KeyCode.S) then moveInput = moveInput + Vector3.new(0, 0, 1) end
	if userInputService:IsKeyDown(Enum.KeyCode.A) then moveInput = moveInput + Vector3.new(-1, 0, 0) end
	if userInputService:IsKeyDown(Enum.KeyCode.D) then moveInput = moveInput + Vector3.new(1, 0, 0) end
	if userInputService:IsKeyDown(Enum.KeyCode.Space) then moveInput = moveInput + Vector3.new(0, 1, 0) end
	if userInputService:IsKeyDown(Enum.KeyCode.LeftShift) then moveInput = moveInput + Vector3.new(0, -1, 0) end

	-- Нормализация вектора
	if moveInput.Magnitude > 0 then
		moveInput = moveInput.Unit
	end

	-- Преобразование в мировые координаты
	local forward = cam.CFrame.LookVector * -moveInput.Z
	local right = cam.CFrame.RightVector * moveInput.X
	local up = Vector3.new(0, 1, 0) * moveInput.Y
	local moveDirection = (forward + right + up)

	if moveDirection.Magnitude > 0 then
		moveDirection = moveDirection.Unit
		lastMoveDir = lastMoveDir:Lerp(moveDirection, dt * 10)
		isMoving = true

		-- Включение звука
		if not isSoundPlaying then
			if flySound then flySound:Destroy() end

			flySound = Instance.new("Sound")
			flySound.Name = "FlySound"
			flySound.SoundId = "rbxassetid://596046130"
			flySound.Volume = 5
			flySound.Looped = true
			flySound.Parent = player.Character.HumanoidRootPart

			-- Ожидание загрузки звука
			local soundLoaded = false
			flySound.Loaded:Connect(function()
				soundLoaded = true
				flySound:Play()
				isSoundPlaying = true
			end)

			-- Таймаут на случай проблем с загрузкой
			delay(1, function()
				if not soundLoaded then
					warn("Не удалось загрузить звук полёта")
					if flySound then
						flySound:Destroy()
						flySound = nil
					end
				end
			end)
		end
	else
		lastMoveDir = lastMoveDir * 0.7
		if lastMoveDir.Magnitude < 0.01 then
			lastMoveDir = Vector3.new(0, 0, 0)
			isMoving = false
		end

		-- Выключение звука
		if isSoundPlaying then
			if flySound then
				flySound:Stop()
				flySound:Destroy()
				flySound = nil
			end
			isSoundPlaying = false
		end
	end

	-- Применение скорости
	if bodyVelocity then
		bodyVelocity.Velocity = lastMoveDir * flySpeed
	end

	-- Наклон персонажа
	if bodyGyro then
		if isMoving then
			local cameraRelativeDir = cam.CFrame:VectorToObjectSpace(lastMoveDir)
			local rollAngle = -cameraRelativeDir.X * math.rad(15)
			local pitchAngle = cameraRelativeDir.Z * math.rad(10)

			bodyGyro.CFrame = bodyGyro.CFrame:Lerp(
				CFrame.new(rootPart.Position, rootPart.Position + cam.CFrame.LookVector) * 
					CFrame.Angles(pitchAngle, 0, rollAngle),
				dt * 6
			)
		else
			bodyGyro.CFrame = bodyGyro.CFrame:Lerp(
				CFrame.new(rootPart.Position, rootPart.Position + cam.CFrame.LookVector),
				dt * 4
			)
		end
	end
end
-- Обновление скорости полёта из TextBox
flySpeedBox.FocusLost:Connect(function(enterPressed)
	local newSpeed = tonumber(flySpeedBox.Text)
	if newSpeed and newSpeed > 0 and newSpeed <= 500 then
		flySpeed = newSpeed
		flySpeedBox.PlaceholderText = "Скорость полёта: "..tostring(flySpeed)
		flySpeedBox.Text = ""
	else
		flySpeedBox.Text = ""
		flySpeedBox.PlaceholderText = "Только числа 1-500!"
	end
end)

-- Обновление множителя скорости бега из TextBox
walkSpeedMultiplierBox.FocusLost:Connect(function(enterPressed)
	local newMultiplier = tonumber(walkSpeedMultiplierBox.Text)
	if newMultiplier and newMultiplier > 0 and newMultiplier <= 10 then
		walkSpeedMultiplier = newMultiplier
		walkSpeedMultiplierBox.PlaceholderText = "Множитель скорости: "..tostring(walkSpeedMultiplier)
		walkSpeedMultiplierBox.Text = ""

		-- Применить новый множитель, если режим скорости активен
		if speedEnabled then
			local humanoid = player.Character and player.Character:FindFirstChild("Humanoid")
			if humanoid then
				humanoid.WalkSpeed = baseWalkSpeed * walkSpeedMultiplier
			end
		end
	else
		walkSpeedMultiplierBox.Text = ""
		walkSpeedMultiplierBox.PlaceholderText = "Только числа 0.1-10!"
	end
end)

-- Кнопки (Fly, Speed, Noclip, Teleport)
local buttons = {
	{Name = "Полёт", Position = 0.2},
	{Name = "Скорость", Position = 0.26},
	{Name = "Noclip", Position = 0.32},
	{Name = "Teleport Tool", Position = 0.38},
	{Name = "Телепорт", Position = 0.44}
}

for _, buttonInfo in pairs(buttons) do
	local button = Instance.new("TextButton")
	button.Name = buttonInfo.Name
	button.Text = buttonInfo.Name
	button.Size = UDim2.new(0.8, 0, 0.05, 0)
	button.Position = UDim2.new(0.1, 0, buttonInfo.Position, 0)
	button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
	button.TextColor3 = Color3.fromRGB(255, 255, 255)
	button.Parent = mainFrame

	button.MouseButton1Click:Connect(function()
		if buttonInfo.Name == "Полёт" then
			flyEnabled = not flyEnabled
			local humanoid = player.Character and player.Character:FindFirstChild("Humanoid")

			if flyEnabled then
				if humanoid then
					humanoid.PlatformStand = true
					bodyVelocity = Instance.new("BodyVelocity")
					bodyVelocity.Velocity = Vector3.new(0, 0, 0)
					bodyVelocity.MaxForce = Vector3.new(10000, 10000, 10000)
					bodyVelocity.Parent = player.Character.HumanoidRootPart

					bodyGyro = Instance.new("BodyGyro")
					bodyGyro.MaxTorque = Vector3.new(10000, 10000, 10000)
					bodyGyro.P = 1000
					bodyGyro.D = 50
					bodyGyro.Parent = player.Character.HumanoidRootPart

					flyConnection = runService.Heartbeat:Connect(updateFly)
				end
			else
				if humanoid then
					humanoid.PlatformStand = false
					if bodyVelocity then bodyVelocity:Destroy() end
					if bodyGyro then bodyGyro:Destroy() end
					if flyConnection then flyConnection:Disconnect() end
					currentVelocity = Vector3.new(0, 0, 0)
					isCameraSmooth = false
				end
			end

		elseif buttonInfo.Name == "Скорость" then
			speedEnabled = not speedEnabled
			local humanoid = player.Character and player.Character:FindFirstChild("Humanoid")

			if humanoid then
				humanoid.WalkSpeed = speedEnabled and (baseWalkSpeed * walkSpeedMultiplier) or baseWalkSpeed
			end

		elseif buttonInfo.Name == "Noclip" then
			noclipEnabled = not noclipEnabled
			button.BackgroundColor3 = noclipEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(50, 50, 50)

			if noclipEnabled then
				noclip()
				noclipConnection = runService.Heartbeat:Connect(function()
					if player.Character then
						for _, part in pairs(player.Character:GetDescendants()) do
							if part:IsA("BasePart") then
								part.CanCollide = false
							end
						end
					end
				end)
			else
				if noclipConnection then noclipConnection:Disconnect() end
				if player.Character then
					for _, part in pairs(player.Character:GetDescendants()) do
						if part:IsA("BasePart") then
							part.CanCollide = true
						end
					end
				end
			end

		elseif buttonInfo.Name == "Teleport Tool" then
			local oldTool = player.Backpack:FindFirstChild("TeleportTool") or player.Character:FindFirstChild("TeleportTool")
			if oldTool then oldTool:Destroy() end

			local tool = Instance.new("Tool")
			tool.Name = "TeleportTool"
			tool.RequiresHandle = false
			tool.Parent = player.Backpack

			local targetPart = Instance.new("Part")
			targetPart.Size = Vector3.new(2, 0.2, 2)
			targetPart.Anchored = true
			targetPart.CanCollide = false
			targetPart.Transparency = 0.7
			targetPart.Color = Color3.fromRGB(0, 255, 0)
			targetPart.Parent = nil

			tool.Equipped:Connect(function()
				targetPart.Parent = workspace

				local connection
				connection = runService.Heartbeat:Connect(function()
					local ray = Ray.new(
						player.Character.HumanoidRootPart.Position,
						player.Character.HumanoidRootPart.CFrame.LookVector * 1000
					)
					local hit, position = workspace:FindPartOnRayWithIgnoreList(ray, {player.Character, targetPart})

					if position then
						targetPart.Position = position
						targetPart.CFrame = CFrame.new(position) * CFrame.Angles(math.pi/2, 0, 0)
					end
				end)

				tool.Unequipped:Connect(function()
					connection:Disconnect()
					targetPart.Parent = nil
				end)
			end)

			tool.Activated:Connect(function()
				if targetPart.Parent then
					player.Character:SetPrimaryPartCFrame(CFrame.new(targetPart.Position + Vector3.new(0, 3, 0)))
					targetPart.Parent = nil
				end
			end)

		elseif buttonInfo.Name == "Телепорт" then
			-- Создаем фрейм для списка игроков
			local playersFrame = Instance.new("Frame")
			playersFrame.Name = "PlayersFrame"
			playersFrame.Size = UDim2.new(0.9, 0, 0.45, 0)
			playersFrame.Position = UDim2.new(0.05, 0, 0.5, 0)
			playersFrame.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
			playersFrame.BackgroundTransparency = 0.3
			playersFrame.BorderSizePixel = 0
			playersFrame.Parent = mainFrame

			local scrollFrame = Instance.new("ScrollingFrame")
			scrollFrame.Size = UDim2.new(1, 0, 0.85, 0)
			scrollFrame.Position = UDim2.new(0, 0, 0.15, 0)
			scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
			scrollFrame.ScrollBarThickness = 5
			scrollFrame.BackgroundTransparency = 1
			scrollFrame.Parent = playersFrame

			local layout = Instance.new("UIListLayout")
			layout.Parent = scrollFrame

			-- Радиокнопки для выбора позиции телепорта
			local frontRadio = Instance.new("TextButton")
			frontRadio.Name = "FrontRadio"
			frontRadio.Text = "Телепорт спереди"
			frontRadio.Size = UDim2.new(0.45, 0, 0.1, 0)
			frontRadio.Position = UDim2.new(0.05, 0, 0.03, 0)
			frontRadio.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
			frontRadio.TextColor3 = Color3.fromRGB(255, 255, 255)
			frontRadio.Parent = playersFrame

			local backRadio = Instance.new("TextButton")
			backRadio.Name = "BackRadio"
			backRadio.Text = "Телепорт сзади"
			backRadio.Size = UDim2.new(0.45, 0, 0.1, 0)
			backRadio.Position = UDim2.new(0.5, 0, 0.03, 0)
			backRadio.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
			backRadio.TextColor3 = Color3.fromRGB(255, 255, 255)
			backRadio.Parent = playersFrame

			frontRadio.MouseButton1Click:Connect(function()
				teleportOffset = 3
				frontRadio.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
				backRadio.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
			end)

			backRadio.MouseButton1Click:Connect(function()
				teleportOffset = -3
				backRadio.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
				frontRadio.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
			end)

			-- Закрытие фрейма по кнопке
			local closeButton = Instance.new("TextButton")
			closeButton.Name = "CloseButton"
			closeButton.Text = "X"
			closeButton.Size = UDim2.new(0.1, 0, 0.1, 0)
			closeButton.Position = UDim2.new(0.9, 0, 0.03, 0)
			closeButton.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
			closeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
			closeButton.Parent = playersFrame

			closeButton.MouseButton1Click:Connect(function()
				playersFrame:Destroy()
			end)

			-- Добавляем игроков в список
			local function updatePlayerList()
				scrollFrame:ClearAllChildren()

				for _, plr in pairs(players:GetPlayers()) do
					if plr ~= player and plr.Character then
						local playerButton = Instance.new("TextButton")
						playerButton.Name = plr.Name
						playerButton.Text = plr.Name
						playerButton.Size = UDim2.new(1, -10, 0, 30)
						playerButton.Position = UDim2.new(0, 5, 0, 0)
						playerButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
						playerButton.TextColor3 = Color3.fromRGB(255, 255, 255)
						playerButton.Parent = scrollFrame

						playerButton.MouseButton1Click:Connect(function()
							if plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
								local tpSound = Instance.new("Sound")
								tpSound.Name = "FlySound"
								tpSound.SoundId = "rbxassetid://5066021887"
								tpSound.Volume = 5
								tpSound.Looped = false
								tpSound.Parent = player.Character.HumanoidRootPart
								game.Debris:AddItem(tpSound, 2)
								player.Character:SetPrimaryPartCFrame(
									plr.Character.HumanoidRootPart.CFrame * CFrame.new(0, 0, teleportOffset))
							end
							playersFrame:Destroy()
						end)
					end
				end

				-- Обновляем размер CanvasSize
				scrollFrame.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y)
			end

			updatePlayerList()

			-- Обновляем список при подключении/отключении игроков
			local playersChangedConn
			playersChangedConn = players.PlayerAdded:Connect(updatePlayerList)
			players.PlayerRemoving:Connect(updatePlayerList)

			playersFrame.Destroying:Connect(function()
				playersChangedConn:Disconnect()
			end)
		end
	end)
end

-- Перетаскивание меню
local dragging = false
local dragStartPos, frameStartPos

title.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStartPos = Vector2.new(input.Position.X, input.Position.Y)
		frameStartPos = mainFrame.Position
	end
end)

title.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = false
	end
end)

userInputService.InputChanged:Connect(function(input)
	if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
		local dragDelta = Vector2.new(input.Position.X, input.Position.Y) - dragStartPos
		mainFrame.Position = UDim2.new(
			frameStartPos.X.Scale,
			frameStartPos.X.Offset + dragDelta.X,
			frameStartPos.Y.Scale,
			frameStartPos.Y.Offset + dragDelta.Y
		)
	end
end)

-- Кнопка сворачивания/разворачивания
local toggleButton = Instance.new("TextButton")
toggleButton.Name = "ToggleButton"
toggleButton.Text = "▼"
toggleButton.Size = UDim2.new(0.1, 0, 0.05, 0)
toggleButton.Position = UDim2.new(0.9, 0, 0.95, 0)
toggleButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
toggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleButton.Parent = mainFrame

local isMinimized = false

toggleButton.MouseButton1Click:Connect(function()
	isMinimized = not isMinimized

	local targetSize = isMinimized and UDim2.new(0.3, 0, 0.1, 0) or UDim2.new(0.3, 0, 0.7, 0)
	local tween = TweenService:Create(mainFrame, tweenInfo, {Size = targetSize})
	tween:Play()

	toggleButton.Text = isMinimized and "▲" or "▼"
end)

-- Горячая клавиша F5 для меню
local menuVisible = true
userInputService.InputBegan:Connect(function(input, processed)
	if not processed and input.KeyCode == Enum.KeyCode.F5 then
		menuVisible = not menuVisible
		gui.Enabled = menuVisible
	end
end)
sendToDiscord(string.format("FlyMod полностью инициализирован для %s", player.Name))
