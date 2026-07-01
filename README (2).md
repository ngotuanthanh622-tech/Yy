-- LocalScript trong StarterPlayer > StarterPlayerScripts

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

-- Giả định bạn sẽ tạo một RemoteEvent trong ReplicatedStorage để báo cho Server biết khi nào đấm/lướt
-- (Nếu chưa tạo RemoteEvent này game vẫn chạy nhưng chỉ có hiệu ứng ở máy bạn)
local CombatEvent = ReplicatedStorage:FindFirstChild("CombatEvent") 

-- Cấu hình Combo Đấm
local lastAttackTime = 0
local comboIndex = 1
local MAX_COMBO = 3
local COMBO_COOLDOWN = 0.8 -- Thời gian hồi giữa các combo

-- Cấu hình Dash (Lướt)
local canDash = true
local DASH_COOLDOWN = 1.5
local DASH_SPEED = 60

-- Hàm xử lý Combo Đấm (Chuột trái)
local function executeCombat()
	local currentTime = tick()
	
	-- Nếu bấm quá nhanh sau khi hết combo thì không tính
	if currentTime - lastAttackTime < 0.3 then return end
	
	-- Nếu để quá lâu không bấm, combo tự reset về 1
	if currentTime - lastAttackTime > COMBO_COOLDOWN then
		comboIndex = 1
	end
	
	print("Client: Thực hiện đấm Combo phát thứ: " .. comboIndex)
	
	-- Gửi tín hiệu lên Server để Server tạo hiệu ứng gây sát thương (Animation, Hitbox)
	if CombatEvent then
		CombatEvent:FireServer("Punch", comboIndex)
	end
	
	-- Chuyển sang phát đấm tiếp theo
	lastAttackTime = currentTime
	if comboIndex < MAX_COMBO then
		comboIndex = comboIndex + 1
	else
		comboIndex = 1 -- Reset nếu đã đấm hết chuỗi combo
	end
end

-- Hàm xử lý Lướt (Dash - Phím Q)
local function executeDash()
	if not canDash or humanoid.MoveDirection == Vector3.new(0,0,0) then return end
	canDash = false
	
	print("Client: Đang lướt (Dash)!")
	
	-- Lấy hướng di chuyển hiện tại của người chơi
	local moveDirection = humanoid.MoveDirection
	
	-- Tạo lực đẩy vật lý (LinearVelocity) để đẩy nhân vật lướt đi
	local attachment = Instance.new("Attachment")
	attachment.Parent = rootPart
	
	local linearVelocity = Instance.new("LinearVelocity")
	linearVelocity.MaxForce = 50000
	linearVelocity.VectorVelocity = moveDirection * DASH_SPEED
	linearVelocity.Attachment0 = attachment
	linearVelocity.Parent = rootPart
	
	-- Gửi tín hiệu lên Server để người chơi khác cũng thấy mình đang lướt (tùy chọn)
	if CombatEvent then
		CombatEvent:FireServer("Dash")
	end
	
	-- Xóa lực đẩy sau 0.2 giây (Thời gian lướt ngắn)
	task.wait(0.2)
	linearVelocity:Destroy()
	attachment:Destroy()
	
	-- Thời gian hồi chiêu lướt
	task.wait(DASH_COOLDOWN)
	canDash = true
	print("Client: Dash đã hồi chiêu xong!")
end

-- Lắng nghe tương tác từ người chơi
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end -- Bỏ qua nếu đang chat hoặc mở menu
	
	-- Click chuột trái để Đấm
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		executeCombat()
	
	-- Bấm phím Q để Lướt (Dash)
	elseif input.KeyCode == Enum.KeyCode.Q then
		executeDash()
	end
end)

-- Cập nhật lại nhân vật khi bị reset/hồi sinh
player.CharacterAdded:Connect(function(newCharacter)
	character = newCharacter
	humanoid = newCharacter:WaitForChild("Humanoid")
	rootPart = newCharacter:WaitForChild("HumanoidRootPart")
end)
