--=====================================================================
--  AIRS AI AGENT  v2.0.0   (Roblox)
--  Clean Claude-style UI (light, no neon) · Blue/White accent
--  Router: Pollinations AI  ·  API token manager  ·  Personalization
--  Responsive: desktop + mobile (drawer sidebar, touch targets)
--=====================================================================

--========================= SERVICES ==================================
local Players     = game:GetService("Players")
local UIS         = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local RunService  = game:GetService("RunService")
local TweenService= game:GetService("TweenService")
local StarterGui  = game:GetService("StarterGui")
local LocalPlayer = Players.LocalPlayer
local Camera      = workspace.CurrentCamera

--========================= FILESYSTEM ================================
local FS = {}
FS.write  = (type(writefile)  == "function") and writefile  or nil
FS.read   = (type(readfile)   == "function") and readfile   or nil
FS.exists = (type(isfile)     == "function") and isfile     or nil
FS.mkdir  = (type(makefolder) == "function") and makefolder or nil
FS.FILE   = "AIRS_AI_Agent.json"

--========================= CONFIG ====================================
local PRESETS = {
	Balanced = "You are a helpful, thoughtful assistant. Be clear, warm and direct. Answer the question asked, then offer brief extra context only if it genuinely helps.",
	Concise  = "You are a concise assistant. Answer in as few words as possible while staying complete. Prefer short sentences and tight structure. No filler.",
	Detailed = "You are a thorough assistant. Give comprehensive, well-structured answers with reasoning, edge cases and examples where useful.",
	Expert   = "You are a precise technical expert. Use correct terminology, state assumptions, flag uncertainty honestly, and avoid hedging or marketing language.",
	Coder    = "You are a senior software engineer. Lead with working code, then a short explanation of key decisions and pitfalls. Prefer idiomatic, complete, runnable solutions.",
	Tutor    = "You are a patient tutor. Explain step by step, check understanding, build from fundamentals to detail, and use small concrete examples.",
	Creative = "You are an imaginative creative partner. Offer original angles, vivid phrasing and multiple distinct directions rather than one safe answer.",
	Custom   = "",
}

local DefaultConfig = {
	assistantName = "AIRS",
	userName      = "User",
	personality   = "Balanced",
	systemPrompt  = PRESETS.Balanced,
	extraNotes    = "",
	language      = "auto",
	temperature   = 0.7,
	maxTokens     = 2048,
	contextTurns  = 16,
	rememberContext = true,

	model     = "openai",
	models    = { "openai", "openai-fast", "openai-large", "mistral",
	              "llama", "qwen-coder", "deepseek", "gemini" },
	endpoint  = "https://text.pollinations.ai/openai",
	modelListUrl = "https://text.pollinations.ai/models",
	imageUrl  = "https://image.pollinations.ai/prompt/",

	tokens        = {},      -- { { label = "...", value = "...", enabled = true } }
	activeToken   = 0,       -- 1-based index, 0 = none
	rotateOnError = true,
	allowAnonymous = true,

	bulletPoints  = false,
	codeExamples  = true,
	showReasoning = false,
	autoScroll    = true,
	sendEnter     = true,
}

local Config = {}
for k, v in pairs(DefaultConfig) do
	if type(v) == "table" then
		local copy = {}
		for k2, v2 in pairs(v) do copy[k2] = v2 end
		Config[k] = copy
	else
		Config[k] = v
	end
end

--========================= PERSISTENCE ===============================
local function saveConfig()
	if not FS.write then return false end
	local ok = pcall(function()
		FS.write(FS.FILE, HttpService:JSONEncode({ config = Config, convos = AIRS and AIRS.convos or {} }))
	end)
	return ok
end

local function loadConfig()
	if not (FS.read and FS.exists) then return end
	if not FS.exists(FS.FILE) then return end
	local ok, data = pcall(function() return HttpService:JSONDecode(FS.read(FS.FILE)) end)
	if not ok or type(data) ~= "table" then return end
	if type(data.config) == "table" then
		for k, v in pairs(data.config) do
			if Config[k] ~= nil then Config[k] = v end
		end
	end
	return data.convos
end

--========================= THEME =====================================
-- Claude-like: warm neutrals, one calm blue accent, no glow, no neon.
local Theme = {
	accent       = Color3.fromRGB(37, 99, 235),
	accentHover  = Color3.fromRGB(29, 78, 216),
	accentSoft   = Color3.fromRGB(219, 234, 254),

	bg           = Color3.fromRGB(252, 252, 251),
	panel        = Color3.fromRGB(255, 255, 255),
	sidebar      = Color3.fromRGB(247, 247, 245),
	hover        = Color3.fromRGB(240, 240, 238),
	field        = Color3.fromRGB(250, 250, 249),

	border       = Color3.fromRGB(229, 229, 225),
	borderStrong = Color3.fromRGB(209, 209, 204),

	text         = Color3.fromRGB(28, 25, 23),
	textSoft     = Color3.fromRGB(87, 83, 78),
	textMuted    = Color3.fromRGB(138, 133, 127),

	userBubble   = Color3.fromRGB(240, 240, 238),
	ok           = Color3.fromRGB(34, 160, 95),
	warn         = Color3.fromRGB(200, 140, 20),
	err          = Color3.fromRGB(211, 71, 71),

	font         = Enum.Font.Gotham,
	fontMed      = Enum.Font.GothamMedium,
	fontBold     = Enum.Font.GothamBold,
}

--========================= UI HELPERS ================================
local function new(class, props, children)
	local inst = Instance.new(class)
	local parent
	for k, v in pairs(props or {}) do
		if k == "Parent" then parent = v else inst[k] = v end
	end
	for _, c in ipairs(children or {}) do c.Parent = inst end
	if parent then inst.Parent = parent end
	return inst
end

local function corner(parent, radius)
	return new("UICorner", { CornerRadius = UDim.new(0, radius or 8), Parent = parent })
end

local function stroke(parent, color, thickness, transparency)
	return new("UIStroke", {
		Color = color or Theme.border,
		Thickness = thickness or 1,
		Transparency = transparency or 0,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		Parent = parent,
	})
end

local function pad(parent, t, b, l, r)
	return new("UIPadding", {
		PaddingTop = UDim.new(0, t or 0), PaddingBottom = UDim.new(0, b or 0),
		PaddingLeft = UDim.new(0, l or 0), PaddingRight = UDim.new(0, r or 0),
		Parent = parent,
	})
end

local function vlist(parent, spacing, padTop, padBot)
	return new("UIListLayout", {
		Padding = UDim.new(0, spacing or 6),
		SortOrder = Enum.SortOrder.LayoutOrder,
		FillDirection = Enum.FillDirection.Vertical,
		HorizontalAlignment = Enum.HorizontalAlignment.Left,
		Parent = parent,
	})
end

--========================= HTTP LAYER ================================
local function httpRequest(opts)
	local reqFn = (syn and syn.request) or (http and http.request) or http_request or request
	if reqFn then
		local ok, res = pcall(reqFn, {
			Url = opts.Url, Method = opts.Method or "GET",
			Headers = opts.Headers, Body = opts.Body,
		})
		if ok and res then
			return res.StatusCode or 200, res.Body or "", true
		end
		return nil, "request() error: " .. tostring(res), false
	end

	local ok, body = pcall(function()
		if (opts.Method or "GET"):upper() == "POST" then
			return HttpService:PostAsync(opts.Url, opts.Body, Enum.HttpContentType.ApplicationJson)
		end
		return HttpService:GetAsync(opts.Url)
	end)
	if ok then return 200, body, false end
	return nil, "HttpService error: " .. tostring(body), false
end

local function jsonDecode(s)
	local ok, r = pcall(function() return HttpService:JSONDecode(s) end)
	return ok and r or nil
end

--===================== TOKEN / KEY MANAGER ===========================
local Keys = {}

function Keys.list() return Config.tokens end

function Keys.active()
	local t = Config.tokens[Config.activeToken]
	if t and t.value ~= "" then return t.value end
	return nil
end

function Keys.add(label, value)
	value = (value or ""):gsub("%s+", "")
	if value == "" then return false, "empty token" end
	table.insert(Config.tokens, {
		label = (label ~= "" and label) or ("Token " .. #Config.tokens + 1),
		value = value, enabled = true,
	})
	if Config.activeToken == 0 then Config.activeToken = #Config.tokens end
	saveConfig()
	return true
end

function Keys.remove(index)
	if Config.tokens[index] then
		table.remove(Config.tokens, index)
		if Config.activeToken > #Config.tokens then Config.activeToken = #Config.tokens end
		if Config.activeToken < 0 then Config.activeToken = 0 end
		saveConfig()
	end
end

function Keys.setActive(index)
	Config.activeToken = index or 0
	saveConfig()
end

-- Ordered token attempt list: active first, then enabled ones, then nil (anonymous)
function Keys.order()
	local out, seen = {}, {}
	local a = Config.tokens[Config.activeToken]
	if a and a.value ~= "" and a.enabled ~= false then
		table.insert(out, a.value); seen[a.value] = true
	end
	if Config.rotateOnError then
		for _, t in ipairs(Config.tokens) do
			if t.value ~= "" and t.enabled ~= false and not seen[t.value] then
				table.insert(out, t.value); seen[t.value] = true
			end
		end
	end
	if Config.allowAnonymous or #out == 0 then table.insert(out, nil) end
	return out
end

--===================== AIRS CORE (ROUTER) ============================
local AIRS = { convos = {}, current = nil, cancel = false, busy = false }

local function buildSystem()
	local name = Config.assistantName ~= "" and Config.assistantName or "AIRS"
	local user = Config.userName ~= "" and Config.userName or "User"
	local base = (Config.systemPrompt ~= "" and Config.systemPrompt) or PRESETS.Balanced

	local lines = { base }
	lines[#lines + 1] = ("Your name is %s. The person you are talking to is %s."):format(name, user)
	if Config.bulletPoints then lines[#lines + 1] = "Prefer short bullet points over long paragraphs." end
	if Config.codeExamples then lines[#lines + 1] = "Include minimal runnable code examples when they clarify the answer." end
	if Config.showReasoning then lines[#lines + 1] = "Briefly explain your reasoning before the final answer." end
	if Config.language ~= "auto" and Config.language ~= "" then
		lines[#lines + 1] = "Always respond in " .. Config.language .. "."
	end
	if Config.extraNotes ~= "" then lines[#lines + 1] = "Extra user instructions:\n" .. Config.extraNotes end
	return table.concat(lines, "\n")
end

-- Single completion attempt
function AIRS._call(messages, model, token, temperature)
	local headers = { ["Content-Type"] = "application/json" }
	local url = Config.endpoint
	if token then headers["Authorization"] = "Bearer " .. token end

	local body = HttpService:JSONEncode({
		model = model,
		messages = messages,
		temperature = temperature or Config.temperature,
		max_tokens = tonumber(Config.maxTokens) or 2048,
		seed = math.random(1, 999999),
	})

	local status, raw = httpRequest({ Url = url, Method = "POST", Headers = headers, Body = body })
	if not raw then return nil, "transport failure", 0 end

	local decoded = jsonDecode(raw)
	if not decoded then
		if status == 200 and #raw > 0 then return raw, nil, status end
		return nil, ("HTTP %s: %s"):format(tostring(status), tostring(raw):sub(1, 160)), status
	end

	if decoded.choices and decoded.choices[1] then
		local c = decoded.choices[1]
		local msg = c.message or c
		local content = msg.content
		if type(content) == "table" then
			local acc = {}
			for _, part in ipairs(content) do
				if type(part) == "table" and part.text then acc[#acc + 1] = part.text
				elseif type(part) == "string" then acc[#acc + 1] = part end
			end
			content = table.concat(acc, "")
		end
		if content and #content > 0 then return content, nil, status end
	end

	if decoded.error then
		local e = decoded.error
		local m = type(e) == "table" and (e.message or "provider error") or tostring(e)
		return nil, m, status
	end
	return nil, "empty response", status
end

-- Router with model fallback + token rotation
function AIRS.ask(prompt, onStage)
	AIRS.cancel = false
	local convo = AIRS.current
	if not convo then return nil, "no conversation" end

	table.insert(convo.messages, { role = "user", content = prompt, ts = os.time() })

	local messages = { { role = "system", content = buildSystem() } }
	if Config.rememberContext then
		local start = math.max(1, #convo.messages - (tonumber(Config.contextTurns) or 16) * 2)
		for i = start, #convo.messages do
			local m = convo.messages[i]
			messages[#messages + 1] = { role = m.role, content = m.content }
		end
	else
		messages[#messages + 1] = { role = "user", content = prompt }
	end

	local modelOrder = { Config.model }
	for _, m in ipairs(Config.models) do
		if m ~= Config.model then modelOrder[#modelOrder + 1] = m end
	end

	local lastErr = "no models available"
	for _, model in ipairs(modelOrder) do
		if AIRS.cancel then table.remove(convo.messages); return nil, "cancelled" end
		if onStage then onStage("Thinking · " .. model) end

		for _, token in ipairs(Keys.order()) do
			if AIRS.cancel then table.remove(convo.messages); return nil, "cancelled" end
			local reply, err, status = AIRS._call(messages, model, token)

			if reply then
				if onStage then onStage("Composing…") end
				local entry = { role = "assistant", content = reply, model = model, ts = os.time() }
				table.insert(convo.messages, entry)
				if #convo.messages == 2 then convo.title = prompt:sub(1, 42) end
				saveConfig()
				return reply, model
			end

			lastErr = ("%s / %s → %s"):format(model, token and "token" or "anon", tostring(err))
			if status == 401 or status == 403 or status == 429 then
				if onStage then onStage("Token rejected · retrying") end
			end
		end
	end

	table.remove(convo.messages)
	return nil, lastErr
end

function AIRS.testConnection()
	local token = Keys.active()
	local headers = {}
	if token then headers["Authorization"] = "Bearer " .. token end
	local status, raw = httpRequest({ Url = Config.modelListUrl, Method = "GET", Headers = headers })
	if not raw then return false, "unreachable" end
	local list = jsonDecode(raw)
	if type(list) == "table" then
		local names = {}
		for _, m in ipairs(list) do
			if type(m) == "string" then names[#names + 1] = m
			elseif type(m) == "table" and m.name then names[#names + 1] = m.name end
		end
		return true, names, status
	end
	return false, "bad response (HTTP " .. tostring(status) .. ")"
end

function AIRS.newConvo()
	local c = { id = HttpService:GenerateGUID(false), title = "New chat", messages = {}, ts = os.time() }
	table.insert(AIRS.convos, 1, c)
	AIRS.current = c
	saveConfig()
	return c
end

--========================= ROOT GUI ==================================
local GUI = new("ScreenGui", {
	Name = "AIRS_AI_Agent",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 100,
})
do
	local ok = pcall(function()
		if gethui then GUI.Parent = gethui()
		elseif syn and syn.protect_gui then syn.protect_gui(GUI); GUI.Parent = game:GetService("CoreGui")
		else GUI.Parent = game:GetService("CoreGui") end
	end)
	if not ok or not GUI.Parent then GUI.Parent = LocalPlayer:WaitForChild("PlayerGui") end
end

--========================= WINDOW ====================================
local Window = new("Frame", {
	Name = "Window",
	Size = UDim2.new(0, 980, 0, 640),
	Position = UDim2.new(0.5, -490, 0.5, -320),
	BackgroundColor3 = Theme.bg,
	BorderSizePixel = 0,
	ClipsDescendants = true,
	Parent = GUI,
})
corner(Window, 12)
stroke(Window, Theme.borderStrong, 1)

--========================= SIDEBAR ===================================
local Sidebar = new("Frame", {
	Name = "Sidebar",
	Size = UDim2.new(0, 214, 1, 0),
	Position = UDim2.new(0, 0, 0, 0),
	BackgroundColor3 = Theme.sidebar,
	BorderSizePixel = 0,
	ZIndex = 4,
	Parent = Window,
})
corner(Sidebar, 12)
new("Frame", { -- square off the right edge of the sidebar
	Size = UDim2.new(0, 12, 1, 0), Position = UDim2.new(1, -12, 0, 0),
	BackgroundColor3 = Theme.sidebar, BorderSizePixel = 0, ZIndex = 4, Parent = Sidebar,
})
pad(Sidebar, 12, 12, 10, 10)

-- Brand
local BrandRow = new("Frame", {
	Size = UDim2.new(1, 0, 0, 34), BackgroundTransparency = 1, LayoutOrder = 1, Parent = Sidebar,
})
new("Frame", {
	Size = UDim2.new(0, 22, 0, 22), Position = UDim2.new(0, 0, 0.5, -11),
	BackgroundColor3 = Theme.accent, BorderSizePixel = 0, Parent = BrandRow,
}) |> function(f) corner(f, 7) end
local BrandDot = BrandRow:FindFirstChildOfClass("Frame")
new("TextLabel", {
	Text = Config.assistantName:sub(1, 1):upper(),
	Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1,
	Font = Theme.fontBold, TextSize = 13, TextColor3 = Color3.new(1, 1, 1), Parent = BrandDot,
})
local BrandLbl = new("TextLabel", {
	Text = Config.assistantName,
	Size = UDim2.new(1, -30, 1, 0), Position = UDim2.new(0, 30, 0, 0),
	BackgroundTransparency = 1, Font = Theme.fontBold, TextSize = 15,
	TextColor3 = Theme.text, TextXAlignment = Enum.TextXAlignment.Left, Parent = BrandRow,
})

-- New chat button
local NewChatBtn = new("TextButton", {
	Text = "  +   New chat",
	Size = UDim2.new(1, 0, 0, 38), LayoutOrder = 2,
	BackgroundColor3 = Theme.panel, BorderSizePixel = 0, AutoButtonColor = false,
	Font = Theme.fontMed, TextSize = 14, TextColor3 = Theme.text,
	TextXAlignment = Enum.TextXAlignment.Left, Parent = Sidebar,
})
corner(NewChatBtn, 8)
stroke(NewChatBtn, Theme.borderStrong, 1)
pad(NewChatBtn, 0, 0, 8, 0)

-- Nav
local NavHolder = new("Frame", {
	Size = UDim2.new(1, 0, 0, 216), BackgroundTransparency = 1, LayoutOrder = 3, Parent = Sidebar,
})
vlist(NavHolder, 2)

-- Conversations
local ConvoHeader = new("TextLabel", {
	Text = "RECENT", Size = UDim2.new(1, 0, 0, 20), LayoutOrder = 4,
	BackgroundTransparency = 1, Font = Theme.fontBold, TextSize = 11,
	TextColor3 = Theme.textMuted, TextXAlignment = Enum.TextXAlignment.Left, Parent = Sidebar,
})
local ConvoScroll = new("ScrollingFrame", {
	Size = UDim2.new(1, 0, 1, -330), LayoutOrder = 5,
	BackgroundTransparency = 1, BorderSizePixel = 0,
	ScrollBarThickness = 3, ScrollBarImageColor3 = Theme.borderStrong,
	CanvasSize = UDim2.new(0, 0, 0, 0), AutomaticCanvasSize = Enum.AutomaticSize.Y,
	Parent = Sidebar,
})
vlist(ConvoScroll, 3)

--========================= CONTENT ===================================
local Content = new("Frame", {
	Name = "Content",
	Size = UDim2.new(1, -214, 1, 0),
	Position = UDim2.new(0, 214, 0, 0),
	BackgroundColor3 = Theme.bg, BorderSizePixel = 0, Parent = Window,
})

local Header = new("Frame", {
	Size = UDim2.new(1, 0, 0, 54), BackgroundColor3 = Theme.panel,
	BorderSizePixel = 0, Parent = Content,
})
stroke(Header, Theme.border, 1, 0)
new("Frame", { -- bottom border only
	Size = UDim2.new(1, 0, 0, 1), Position = UDim2.new(0, 0, 1, -1),
	BackgroundColor3 = Theme.border, BorderSizePixel = 0, Parent = Header,
})
pad(Header, 0, 0, 14, 14)

local Burger = new("TextButton", {
	Text = "☰", Size = UDim2.new(0, 32, 0, 32), Position = UDim2.new(0, 0, 0.5, -16),
	BackgroundTransparency = 1, Font = Theme.fontBold, TextSize = 18,
	TextColor3 = Theme.textSoft, Visible = false, Parent = Header,
})
local TitleLbl = new("TextLabel", {
	Text = "New chat",
	Size = UDim2.new(1, -260, 1, 0), Position = UDim2.new(0, 40, 0, 0),
	BackgroundTransparency = 1, Font = Theme.fontBold, TextSize = 15,
	TextColor3 = Theme.text, TextXAlignment = Enum.TextXAlignment.Left, Parent = Header,
})
local ModelChip = new("TextButton", {
	Text = Config.model .. "  ▾",
	Size = UDim2.new(0, 150, 0, 30), Position = UDim2.new(1, -162, 0.5, -15),
	BackgroundColor3 = Theme.field, BorderSizePixel = 0, AutoButtonColor = false,
	Font = Theme.fontMed, TextSize = 12, TextColor3 = Theme.textSoft, Parent = Header,
})
corner(ModelChip, 8)
stroke(ModelChip, Theme.border, 1)

-- Status pulse dot in header
local StatusDot = new("Frame", {
	Size = UDim2.new(0, 7, 0, 7), Position = UDim2.new(1, -174, 0.5, -3.5),
	BackgroundColor3 = Theme.ok, BorderSizePixel = 0, Parent = Header,
})
corner(StatusDot, 7)

-- Page holder
local PageHolder = new("Frame", {
	Size = UDim2.new(1, 0, 1, -54), Position = UDim2.new(0, 0, 0, 54),
	BackgroundTransparency = 1, Parent = Content,
}
