--!nonstrict
-- © 2026 Nameless Admin. All rights reserved. Do not copy, paste, redistribute, or claim as your own.

local __NARootHost = (getgenv and getgenv()) or _G or {}
local __NARootPreviousNACaller = type(__NARootHost) == "table" and rawget(__NARootHost, "NACaller") or nil
local __NARootErrorState = type(__NARootHost) == "table" and rawget(__NARootHost, "__NAErrorLogState") or nil
if type(__NARootErrorState) ~= "table" then
	__NARootErrorState = { lastIndex = 0; rootCount = 0 }
	if type(__NARootHost) == "table" then
		pcall(rawset, __NARootHost, "__NAErrorLogState", __NARootErrorState)
	end
end

local function __NARootExecutorInfo()
	local name = "Unknown"
	local versionValue = "Unknown"
	if type(identifyexecutor) == "function" then
		local ok, a, b = pcall(identifyexecutor)
		if ok then
			if a ~= nil and tostring(a) ~= "" then name = tostring(a) end
			if b ~= nil and tostring(b) ~= "" then versionValue = tostring(b) end
		end
	end
	if name == "Unknown" and type(getexecutorname) == "function" then
		local ok, value = pcall(getexecutorname)
		if ok and value ~= nil and tostring(value) ~= "" then name = tostring(value) end
	end
	if versionValue == "Unknown" and type(getexecutorversion) == "function" then
		local ok, value = pcall(getexecutorversion)
		if ok and value ~= nil and tostring(value) ~= "" then versionValue = tostring(value) end
	end
	return name, versionValue
end

local function __NARootDebugInfo(fn)
	local source = "Unknown"
	local line = 0
	local name = "anonymous"
	if type(debug) == "table" and type(debug.info) == "function" and type(fn) == "function" then
		pcall(function()
			local s, l, n = debug.info(fn, "sln")
			if s ~= nil and tostring(s) ~= "" then source = tostring(s) end
			if tonumber(l) then line = tonumber(l) end
			if n ~= nil and tostring(n) ~= "" then name = tostring(n) end
		end)
	end
	return source, line, name
end

local function __NARootNextErrorPath()
	local root = "Nameless-Admin"
	local dir = root.."/ErrorLogs"
	local maxIndex = tonumber(__NARootErrorState.lastIndex) or 0
	if type(makefolder) == "function" then
		pcall(function()
			if type(isfolder) ~= "function" or not isfolder(root) then makefolder(root) end
		end)
		pcall(function()
			if type(isfolder) ~= "function" or not isfolder(dir) then makefolder(dir) end
		end)
	end
	if type(listfiles) == "function" then
		local ok, entries = pcall(listfiles, dir)
		if ok and type(entries) == "table" then
			for _, entry in entries do
				local normalized = tostring(entry):gsub("\\", "/")
				local index = tonumber(normalized:match("NA_Error#(%d+)%.txt$"))
				if index and index > maxIndex then maxIndex = index end
			end
		end
	end
	local nextIndex = maxIndex + 1
	local path = dir.."/NA_Error#"..tostring(nextIndex)..".txt"
	if type(isfile) == "function" then
		while isfile(path) do
			nextIndex += 1
			path = dir.."/NA_Error#"..tostring(nextIndex)..".txt"
		end
	end
	__NARootErrorState.lastIndex = nextIndex
	return path, nextIndex
end

local function __NARootReportError(rawError, tracebackText, context, fn, options)
	options = type(options) == "table" and options or {}
	__NARootErrorState.rootCount = (tonumber(__NARootErrorState.rootCount) or 0) + 1
	local logPath, fileIndex = __NARootNextErrorPath()
	local execName, execVersion = __NARootExecutorInfo()
	local fnSource, fnLine, fnName = __NARootDebugInfo(fn)
	local clientVersion = "Unknown"
	if type(version) == "function" then
		local ok, value = pcall(version)
		if ok and value ~= nil and tostring(value) ~= "" then clientVersion = tostring(value) end
	end
	local timestamp = tostring(os.time and os.time() or "Unknown")
	if type(os.date) == "function" then
		local ok, value = pcall(os.date, "!%Y-%m-%dT%H:%M:%SZ")
		if ok and value then timestamp = tostring(value) end
	end
	local placeName = "Unknown"
	local placeId = "Unknown"
	local gameId = "Unknown"
	local jobId = "Unknown"
	pcall(function()
		placeName = tostring(game.Name or "Unknown")
		placeId = tostring(game.PlaceId or "Unknown")
		gameId = tostring(game.GameId or "Unknown")
		jobId = tostring(game.JobId or "Unknown")
	end)
	local platform = "Unknown"
	pcall(function()
		local uis = game:GetService("UserInputService")
		if uis and type(uis.GetPlatform) == "function" then
			platform = tostring(uis:GetPlatform())
		end
	end)
	local errorId = "NAE-ROOT-"..string.format("%06d", tonumber(fileIndex) or tonumber(__NARootErrorState.rootCount) or 1)
	local lines = {
		"[Nameless Admin Root Error Report]";
		"Error ID: "..errorId;
		"Error File: NA_Error#"..tostring(fileIndex or "Unknown")..".txt";
		"Time (UTC): "..timestamp;
		"Context: "..tostring(context or options.context or "Nameless Admin Main Runtime");
		"Severity: "..string.upper(tostring(options.severity or "fatal"));
		"";
		"Runtime";
		"Executor: "..execName;
		"Executor Version: "..execVersion;
		"Roblox Client: "..clientVersion;
		"Platform: "..platform;
		"NA Source: NA testing.lua";
		"Coverage: Root NACaller";
		"";
		"Session";
		"Place: "..placeName;
		"PlaceId: "..placeId;
		"GameId: "..gameId;
		"JobId: "..jobId;
		"";
		"Callback";
		"Function: "..fnName;
		"Defined At: "..fnSource..":"..tostring(fnLine);
		"";
		"Error";
		tostring(rawError or "Unknown error");
		"";
		"Traceback";
		tostring(tracebackText or rawError or "Unavailable");
	}
	if type(options.details) == "table" then
		lines[#lines + 1] = ""
		lines[#lines + 1] = "Additional Context"
		for key, value in options.details do
			lines[#lines + 1] = tostring(key)..": "..tostring(value)
		end
	end
	local report = table.concat(lines, "\n")
	local writer = type(writefile) == "function" and writefile or (type(appendfile) == "function" and appendfile or nil)
	local saved = false
	if options.log ~= false and type(writer) == "function" then
		pcall(function()
			writer(logPath, report.."\n")
			saved = true
		end)
	end
	if options.warn ~= false and type(warn) == "function" then
		pcall(warn, report..(saved and ("\nSaved: "..logPath) or ""))
	end
	return report, saved and logPath or nil, errorId
end

local function __NARootNACaller(fnOrOptions, ...)
	local options = {}
	local fn
	local args
	if type(fnOrOptions) == "table" and type(select(1, ...)) == "function" then
		for key, value in fnOrOptions do options[key] = value end
		fn = select(1, ...)
		args = table.pack(select(2, ...))
	else
		fn = fnOrOptions
		args = table.pack(...)
	end
	if type(fn) ~= "function" then
		local message = "NACaller expected a function, got "..type(fn)
		__NARootReportError(message, message, options.context or "NACaller bootstrap", nil, options)
		return false, message
	end
	local rawError
	local results = table.pack(xpcall(function()
		return fn(table.unpack(args, 1, args.n))
	end, function(message)
		rawError = tostring(message or "Unknown error")
		if type(debug) == "table" and type(debug.traceback) == "function" then
			local ok, trace = pcall(debug.traceback, rawError, 2)
			if ok and type(trace) == "string" and trace ~= "" then return trace end
		end
		return rawError
	end))
	if not results[1] then
		local trace = tostring(results[2] or rawError or "Unknown error")
		__NARootReportError(rawError or trace, trace, options.context or "Nameless Admin Main Runtime", fn, options)
		if options.rethrow == true then error(trace, 0) end
	end
	return table.unpack(results, 1, results.n)
end

if type(__NARootHost) == "table" then
	pcall(rawset, __NARootHost, "NACaller", __NARootNACaller)
end

local __NARootResult = table.pack(NACaller({
	context = "Nameless Admin Main Runtime";
	severity = "fatal";
	warn = true;
	log = true;
}, function()

const _na_boot = {
	hostEnv = (getgenv and getgenv()) or _G or {},
}
_na_boot.hostGetfenv = type(_na_boot.hostEnv.getfenv) == "function" and _na_boot.hostEnv.getfenv or getfenv
_na_boot.hostSetfenv = type(_na_boot.hostEnv.setfenv) == "function" and _na_boot.hostEnv.setfenv or setfenv
_na_boot.hostLoadstring = type(rawget(_na_boot.hostEnv, "loadstring")) == "function" and rawget(_na_boot.hostEnv, "loadstring") or loadstring
_na_boot.hostLoad = type(rawget(_na_boot.hostEnv, "load")) == "function" and rawget(_na_boot.hostEnv, "load") or load
_na_boot.debug = rawget(_na_boot.hostEnv, "debug") or debug
_na_boot.getPrivateRegistry = function()
	local registry
	if type(getreg) == "function" then
		pcall(function()
			registry = getreg()
		end)
	end
	if type(registry) ~= "table" and type(_na_boot.debug) == "table" and type(_na_boot.debug.getregistry) == "function" then
		pcall(function()
			registry = _na_boot.debug.getregistry()
		end)
	end
	if type(registry) ~= "table" then
		registry = _na_boot.hostEnv
	end
	return registry
end
_na_boot.ensureTable = function(host, key)
	local value = type(host) == "table" and rawget(host, key) or nil
	if type(value) ~= "table" then
		value = {}
		if type(host) == "table" then
			pcall(rawset, host, key, value)
			if rawget(host, key) ~= value then
				host[key] = value
			end
		end
	end
	return value
end
_na_boot.privateRegistry = _na_boot.getPrivateRegistry()
_na_boot.privateRoot = _na_boot.ensureTable(_na_boot.privateRegistry, "__nameless_admin_private")
const _na_env = _na_boot.ensureTable(_na_boot.privateRoot, "testing")
const _na_shared = _na_boot.ensureTable(_na_env, "shared")
_na_boot.runtimeEnv = _na_boot.ensureTable(_na_env, "runtime")

_na_boot.runtimeEnv.shared = _na_shared
_na_boot.runtimeEnv._G = _na_boot.runtimeEnv
_na_boot.runtimeEnv.getgenv = function()
	return _na_boot.runtimeEnv
end
_na_boot.runtimeEnv.getfenv = function(target)
	if target == nil or target == 0 then
		return _na_boot.runtimeEnv
	end
	if type(_na_boot.hostGetfenv) == "function" then
		return _na_boot.hostGetfenv(target)
	end
	return _na_boot.runtimeEnv
end

setmetatable(_na_boot.runtimeEnv, {
	__index = function(_, key)
		if key == "_G" then
			return _na_boot.runtimeEnv
		elseif key == "shared" then
			return _na_shared
		elseif key == "getgenv" then
			return _na_boot.runtimeEnv.getgenv
		elseif key == "getfenv" then
			return _na_boot.runtimeEnv.getfenv
		end
		return _na_boot.hostEnv[key]
	end
})

if type(_na_boot.hostSetfenv) == "function" then
	pcall(_na_boot.hostSetfenv, 1, _na_boot.runtimeEnv)
end

const function naAlreadyLoaded()
	if _na_env and (_na_env.ltseverydayyou_NA or _na_env.NA_LOADED) then
		return true
	end
	if _na_shared and (_na_shared.ltseverydayyou_NA or _na_shared.NA_LOADED) then
		return true
	end
	return false
end

_na_boot.installExistingMCPBridge = function()
	local existing
	for _, target in { _na_env, _na_shared, _na_boot.runtimeEnv, _na_boot.hostEnv } do
		if type(target) == "table" then
			existing = existing or rawget(target, "NA_MCP") or rawget(target, "NamelessAdminMCP")
		end
	end

	const options = type(rawget(_na_env, "NA_MCP_OPTIONS")) == "table" and rawget(_na_env, "NA_MCP_OPTIONS") or {}
	if options.allowUIAccess == nil then
		options.allowUIAccess = false
	end
	if options.commandPrediction == nil then
		options.commandPrediction = false
	end
	if options.notifyCommands == nil then
		options.notifyCommands = true
	end
	if options.notifyReads == nil then
		options.notifyReads = true
	end
	options.requireIdentity = true

	if type(existing) == "table"
		and tonumber(existing.version)
		and tonumber(existing.version) >= 2
		and existing.identityRequired == true
		and type(existing.identify) == "function"
		and type(existing.manifest) == "function"
	then
		existing.alreadyLoaded = true
		for _, target in { _na_env, _na_shared, _na_boot.runtimeEnv, _na_boot.hostEnv } do
			if type(target) == "table" then
				pcall(function()
					target.NA_MCP = existing
					target.NamelessAdminMCP = existing
					target.NA_MCP_OPTIONS = options
					target.cmdRun = target.cmdRun or existing.run
					target.RunCommand = target.RunCommand or existing.run
					target.runCommand = target.runCommand or existing.run
				end)
			end
		end
		return true
	end

	const function cleanIdentityText(value, maxLength)
		local text = tostring(value or "")
		text = text:gsub("^%s+", ""):gsub("%s+$", "")
		if #text > (maxLength or 96) then
			text = text:sub(1, maxLength or 96)
		end
		return text
	end

	const function snapshotIdentity()
		const identity = type(options.identity) == "table" and options.identity or nil
		if not identity then
			return nil
		end
		return {
			provider = cleanIdentityText(identity.provider, 64),
			model = cleanIdentityText(identity.model, 96),
			tool = cleanIdentityText(identity.tool, 96),
			client = cleanIdentityText(identity.client, 96),
			version = cleanIdentityText(identity.version, 48),
			sessionId = cleanIdentityText(identity.sessionId, 96),
			displayName = cleanIdentityText(identity.displayName, 96),
			connectedAt = identity.connectedAt,
		}
	end

	const function identityLabel()
		const identity = snapshotIdentity()
		if not identity then
			return ""
		end
		local tool = identity.tool ~= "" and identity.tool or identity.client
		local model = identity.model
		if tool ~= "" and model ~= "" then
			return tool.." | "..model
		elseif model ~= "" then
			return model
		end
		return tool
	end

	const function normalizeIdentity(info)
		if type(info) ~= "table" then
			return nil, "identity must be a table"
		end
		const model = cleanIdentityText(info.model or info.modelName or info.aiModel, 96)
		const tool = cleanIdentityText(info.tool or info.aiTool or info.clientTool or info.mcpTool, 96)
		const client = cleanIdentityText(info.client or info.clientName or info.application, 96)
		if model == "" then
			return nil, "AI model is required"
		end
		if tool == "" then
			return nil, "AI tool is required"
		end
		return {
			provider = cleanIdentityText(info.provider or info.vendor or info.company, 64),
			model = model,
			tool = tool,
			client = client,
			version = cleanIdentityText(info.version or info.clientVersion or info.toolVersion, 48),
			sessionId = cleanIdentityText(info.sessionId or info.session or info.conversationId, 96),
			displayName = cleanIdentityText(info.displayName or info.name, 96),
			connectedAt = tick(),
		}
	end

	const function notifyExistingBridge(action, detail, isRead, force)
		if options.notifyCommands ~= true and force ~= true then
			return
		end
		if isRead and options.notifyReads ~= true and force ~= true then
			return
		end
		local notifier = rawget(_na_env, "DoNotif") or rawget(_na_shared, "DoNotif") or rawget(_na_boot.runtimeEnv, "DoNotif") or rawget(_na_boot.hostEnv, "DoNotif")
		if type(notifier) ~= "function" and type(DoNotif) == "function" then
			notifier = DoNotif
		end
		if type(notifier) ~= "function" then
			return
		end
		const label = identityLabel()
		const fallbackActor = cleanIdentityText(options.actor, 96)
		const actor = label ~= "" and label or fallbackActor
		const prefix = actor ~= "" and ("MCP ["..actor.."]") or "MCP"
		local msg = prefix.." "..tostring(action or "activity")
		if detail and tostring(detail) ~= "" then
			msg ..= ": "..tostring(detail)
		end
		if #msg > 220 then
			msg = msg:sub(1, 217).."..."
		end
		pcall(notifier, msg, isRead and 1.75 or 2.75, "MCP")
	end

	const function findRunner()
		for _, target in { _na_env, _na_shared, _na_boot.runtimeEnv, _na_boot.hostEnv } do
			if type(target) == "table" then
				for _, key in { "RunCommand", "runCommand", "cmdRun" } do
					const fn = rawget(target, key)
					if type(fn) == "function" then
						return fn
					end
				end
			end
		end
		return nil
	end

	const bridge = type(existing) == "table" and existing or {}
	const previousHelpers = {}
	for _, helperName in { "run", "runSequence", "commands", "snapshot", "basicInfo", "logs", "ui", "activity", "options" } do
		const fn = bridge[helperName]
		if type(fn) == "function" then
			previousHelpers[helperName] = fn
		end
	end
	const previousRun = previousHelpers.run
	const previousUI = previousHelpers.ui
	bridge.name = "Nameless Admin MCP Bridge"
	bridge.version = 2
	bridge.protocolVersion = "2.0"
	bridge.kind = "nameless-admin"
	bridge.ready = true
	bridge.alreadyLoaded = true
	bridge.identityRequired = true
	bridge.requiredIdentityFields = { "model", "tool" }
	bridge.instructions = "Before using NA MCP operational helpers, identify yourself with bridge.identify({provider=..., model=..., tool=..., client=...}). Use the exact model name when available and never invent one. Tell the user which AI tool/client and model are connected to Nameless Admin MCP Bridge. After each command or state-changing action, tell the user which NA MCP helper/command ran and summarize its result."
	bridge.helpers = { "identify", "handshake", "hello", "whoami", "manifest", "help", "status", "run", "runSequence", "commands", "snapshot", "basicInfo", "logs", "ui", "activity", "options", "ping", "disconnectAI" }

	const function meta(operation)
		const identity = snapshotIdentity()
		local disclosure
		if identity then
			const provider = identity.provider ~= "" and (" by "..identity.provider) or ""
			disclosure = "Nameless Admin MCP Bridge "..tostring(operation or "activity").." used "..identity.tool.." with "..identity.model..provider.."."
		end
		return {
			bridge = bridge.name,
			bridgeVersion = bridge.version,
			protocolVersion = bridge.protocolVersion,
			operation = tostring(operation or ""),
			identityRequired = true,
			ai = identity,
			disclosureRequired = identity ~= nil,
			userDisclosure = disclosure,
		}
	end

	const function attachMeta(payload, operation)
		if type(payload) ~= "table" then
			payload = { ok = true, result = payload }
		end
		const mcpMeta = meta(operation)
		payload.mcp = mcpMeta
		if mcpMeta.userDisclosure and payload.mustTellUser == nil then
			payload.mustTellUser = mcpMeta.userDisclosure
		end
		return payload
	end

	const function requireIdentity(operation)
		if snapshotIdentity() then
			return true
		end
		const payload = {
			ok = false,
			code = "MCP_AI_IDENTITY_REQUIRED",
			error = "AI identity required before '"..tostring(operation or "operation").."'. Call bridge.identify with the AI model and MCP tool/client first.",
			requiredAction = "identify",
			requiredFields = { "model", "tool" },
			instructions = bridge.instructions,
		}
		payload.mcp = meta(operation)
		notifyExistingBridge("blocked "..tostring(operation or "operation"), "AI model/tool identity required", false, true)
		return false, payload
	end

	const function callPrevious(helperName, isRead, ...)
		local allowed, denial = requireIdentity(helperName)
		if not allowed then
			return denial
		end
		const fn = previousHelpers[helperName]
		if type(fn) ~= "function" then
			notifyExistingBridge(helperName.." error", "helper unavailable", isRead == true, true)
			return attachMeta({ ok = false, error = "Existing NA runtime does not expose '"..helperName.."'. Reload NA fully to install the complete MCP v2 bridge." }, helperName)
		end
		local ok, result = pcall(fn, ...)
		if not ok then
			notifyExistingBridge(helperName.." error", tostring(result), isRead == true, true)
			return attachMeta({ ok = false, error = tostring(result) }, helperName)
		end
		return attachMeta(result, helperName)
	end

	bridge.ping = function()
		return attachMeta({
			ok = true,
			name = bridge.name,
			version = bridge.version,
			protocolVersion = bridge.protocolVersion,
			alreadyLoaded = true,
			testing = _na_env and _na_env.NATestingVer == true or false,
			identityRequired = true,
			identified = snapshotIdentity() ~= nil,
			requiredIdentityFields = bridge.requiredIdentityFields,
			helpers = bridge.helpers,
			instructions = bridge.instructions,
			time = os.time and os.time() or nil,
		}, "ping")
	end

	bridge.manifest = function()
		return attachMeta({
			ok = true,
			name = bridge.name,
			version = bridge.version,
			kind = bridge.kind,
			identityRequired = true,
			requiredIdentityFields = bridge.requiredIdentityFields,
			instructions = bridge.instructions,
			tools = {
				{ name = "identify", description = "Register the AI provider, exact model, MCP tool and client before operational access." },
				{ name = "whoami", description = "Read the currently registered AI identity." },
				{ name = "ping", description = "Read bridge version, readiness and handshake requirements." },
				{ name = "run", description = "Run an exposed Nameless Admin command after AI identification." },
				{ name = "ui", description = "Read NA UI metadata; the Instance is returned only when UI access is enabled." },
				{ name = "options", description = "Read or change bridge options. Mutating options requires identification." },
				{ name = "disconnectAI", description = "Clear the active AI identity and require a new handshake." },
			},
		}, "manifest")
	end
	bridge.help = bridge.manifest

	bridge.identify = function(info)
		local identity, err = normalizeIdentity(info)
		if not identity then
			return attachMeta({
				ok = false,
				code = "MCP_INVALID_AI_IDENTITY",
				error = err,
				requiredFields = { "model", "tool" },
				instructions = bridge.instructions,
			}, "identify")
		end
		options.identity = identity
		options.actor = identityLabel()
		notifyExistingBridge("AI connected", identityLabel(), false, true)
		const provider = identity.provider ~= "" and (" by "..identity.provider) or ""
		return attachMeta({
			ok = true,
			identity = snapshotIdentity(),
			disclosureRequired = true,
			userDisclosure = "Connected to Nameless Admin MCP Bridge through "..identity.tool.." using "..identity.model..provider..".",
			instructions = bridge.instructions,
		}, "identify")
	end
	bridge.handshake = bridge.identify
	bridge.hello = bridge.identify

	bridge.whoami = function()
		const identity = snapshotIdentity()
		return attachMeta({
			ok = identity ~= nil,
			identified = identity ~= nil,
			identity = identity,
			requiredFields = identity and nil or { "model", "tool" },
			instructions = bridge.instructions,
		}, "whoami")
	end

	bridge.status = function()
		return attachMeta({
			ok = true,
			ready = true,
			identified = snapshotIdentity() ~= nil,
			identity = snapshotIdentity(),
			allowUIAccess = options.allowUIAccess == true,
			commandPrediction = options.commandPrediction == true,
			notifyCommands = options.notifyCommands == true,
			notifyReads = options.notifyReads == true,
			requireIdentity = true,
		}, "status")
	end

	bridge.options = function(nextOptions)
		if type(nextOptions) == "table" then
			local allowed, denial = requireIdentity("options")
			if not allowed then
				return denial
			end
			if type(previousHelpers.options) == "function" then
				local ok, result = pcall(previousHelpers.options, nextOptions)
				if not ok then
					return attachMeta({ ok = false, error = tostring(result) }, "options")
				end
				return attachMeta(result, "options")
			end
			const changed = {}
			for key, value in nextOptions do
				if key == "allowUIAccess" or key == "commandPrediction" or key == "notifyCommands" or key == "notifyReads" then
					options[key] = value == true
					changed[#changed + 1] = key.."="..tostring(options[key])
				elseif key == "notifyActivity" then
					options.notifyCommands = value == true
					changed[#changed + 1] = "notifyCommands="..tostring(options.notifyCommands)
				elseif key == "actor" then
					options.actor = cleanIdentityText(value, 96)
				end
			end
			if #changed > 0 then
				notifyExistingBridge("options", table.concat(changed, ", "), false, true)
			end
		elseif type(previousHelpers.options) == "function" then
			local ok, result = pcall(previousHelpers.options)
			if ok then
				return attachMeta(result, "options")
			end
		end
		return attachMeta({
			ok = true,
			allowUIAccess = options.allowUIAccess == true,
			commandPrediction = options.commandPrediction == true,
			notifyCommands = options.notifyCommands == true,
			notifyActivity = options.notifyCommands == true,
			notifyReads = options.notifyReads == true,
			requireIdentity = true,
			identity = snapshotIdentity(),
		}, "options")
	end

	bridge.run = function(...)
		local allowed, denial = requireIdentity("run")
		if not allowed then
			return denial
		end
		if type(previousRun) == "function" and previousRun ~= bridge.run then
			local ok, result = pcall(previousRun, ...)
			if not ok then
				notifyExistingBridge("command error", tostring(result), false, true)
				return attachMeta({ ok = false, error = tostring(result) }, "run")
			end
			return attachMeta(result, "run")
		end
		const runner = findRunner()
		if type(runner) ~= "function" or runner == bridge.run then
			notifyExistingBridge("command error", "runner unavailable", false, true)
			return attachMeta({ ok = false, error = "NA command runner is not exposed yet." }, "run")
		end
		local ok, result = pcall(runner, ...)
		if not ok then
			notifyExistingBridge("command error", tostring(result), false, true)
			return attachMeta({ ok = false, error = tostring(result) }, "run")
		end
		const parts = {}
		for i = 1, select("#", ...) do
			parts[#parts + 1] = tostring(select(i, ...) or "")
		end
		notifyExistingBridge("ran", table.concat(parts, " "), false, false)
		return attachMeta({ ok = true, result = result }, "run")
	end

	bridge.ui = function()
		local allowed, denial = requireIdentity("ui")
		if not allowed then
			return denial
		end
		if type(previousUI) == "function" and previousUI ~= bridge.ui then
			local ok, result = pcall(previousUI)
			if not ok then
				return attachMeta({ ok = false, error = tostring(result) }, "ui")
			end
			return attachMeta(result, "ui")
		end
		notifyExistingBridge("requested UI", options.allowUIAccess == true and "access granted" or "access denied", true, false)
		local gui = rawget(_na_env, "NA_UI_INSTANCE") or rawget(_na_env, "NA_RAW_UI")
			or rawget(_na_shared, "NA_UI_INSTANCE") or rawget(_na_shared, "NA_RAW_UI")
		if not gui then
			const get = rawget(_na_env, "NA_UI") or rawget(_na_shared, "NA_UI")
			if type(get) == "function" then
				pcall(function()
					gui = get()
				end)
			end
		end
		const isInst = typeof(gui) == "Instance"
		const info = {
			ok = true,
			hasUI = isInst,
			allowUIAccess = options.allowUIAccess == true,
		}
		if isInst then
			info.name = gui.Name
			info.className = gui.ClassName
			info.enabled = gui.Enabled
			info.parentClassName = gui.Parent and gui.Parent.ClassName or nil
			if options.allowUIAccess == true then
				info.instance = gui
			end
		end
		return attachMeta(info, "ui")
	end

	bridge.runSequence = function(...)
		return callPrevious("runSequence", false, ...)
	end

	bridge.commands = function(...)
		return callPrevious("commands", true, ...)
	end

	bridge.snapshot = function(...)
		return callPrevious("snapshot", true, ...)
	end

	bridge.basicInfo = function(...)
		return callPrevious("basicInfo", true, ...)
	end

	bridge.logs = function(...)
		return callPrevious("logs", true, ...)
	end

	bridge.activity = function(...)
		return callPrevious("activity", true, ...)
	end

	bridge.disconnectAI = function()
		local allowed, denial = requireIdentity("disconnectAI")
		if not allowed then
			return denial
		end
		const oldIdentity = snapshotIdentity()
		notifyExistingBridge("AI disconnected", identityLabel(), false, true)
		options.identity = nil
		options.actor = ""
		return attachMeta({ ok = true, disconnected = oldIdentity }, "disconnectAI")
	end

	for _, target in { _na_env, _na_shared, _na_boot.runtimeEnv, _na_boot.hostEnv } do
		if type(target) == "table" then
			pcall(function()
				target.NA_MCP = bridge
				target.NamelessAdminMCP = bridge
				target.NA_MCP_OPTIONS = options
				target.cmdRun = target.cmdRun or bridge.run
				target.RunCommand = target.RunCommand or bridge.run
				target.runCommand = target.runCommand or bridge.run
			end)
		end
	end

	return true
end

if naAlreadyLoaded() then
	_na_boot.installExistingMCPBridge()
	return
end

const naFlagValue = tick()
const naVerifyKey = "Arys also known as tim1540 loves skidding and lying like a little bitch he is grow up retard"

_na_boot.syncRuntimeGlobals = function(values)
	if type(values) ~= "table" then
		return
	end

	for _, target in { _na_env, _na_shared, _na_boot.runtimeEnv, _na_boot.hostEnv } do
		if type(target) == "table" then
			for key, value in values do
				pcall(function()
					target[key] = value
				end)
			end
		end
	end
end

_na_boot.syncRuntimeGlobals({
	ltseverydayyou_NA = naFlagValue,
	NA_LOADED = naFlagValue,
	NATestingVer = false,
	NAverify = naVerifyKey,
	NAKey = naVerifyKey,
	__NAKeySource = "Source.lua",
})

_na_boot.lowerHeaders = function(headers)
	const out = {}
	if type(headers) == "table" then
		for key, value in headers do
			out[string.lower(tostring(key))] = value
		end
	end
	return out
end

_na_boot.getResponseStatus = function(response)
	if type(response) ~= "table" then
		return nil
	end
	return tonumber(response.StatusCode or response.statusCode or response.Status or response.status or response.Code or response.code)
end

_na_boot.getResponseBody = function(response)
	if type(response) == "string" then
		return response
	end
	if type(response) ~= "table" then
		return nil
	end
	const body = response.Body or response.body or response.Data or response.data or response.Text or response.text or response.Content or response.content or response.ResponseBody or response.responseBody
	return type(body) == "string" and body or nil
end

_na_boot.getRetryAfter = function(response)
	if type(response) ~= "table" then
		return nil
	end
	const headers = _na_boot.lowerHeaders(response.Headers or response.headers)
	const retryAfter = tonumber(headers["retry-after"]) or tonumber(headers["x-ratelimit-retryafter"]) or tonumber(headers["x-rate-limit-retry-after"])
	return retryAfter
end

_na_boot.isRetryableHttp = function(response, err)
	const status = _na_boot.getResponseStatus(response)
	if status == 408 or status == 425 or status == 429 or (status and status >= 500 and status < 600) then
		return true
	end
	const text = string.lower(tostring(err or _na_boot.getResponseBody(response) or ""))
	return text:find("429", 1, true) ~= nil
		or text:find("too many requests", 1, true) ~= nil
		or text:find("rate limit", 1, true) ~= nil
		or text:find("timed out", 1, true) ~= nil
		or text:find("timeout", 1, true) ~= nil
end

_na_boot.sleepHttp = function(seconds)
	seconds = math.clamp(tonumber(seconds) or 0, 0, 20)
	if type(task) == "table" and type(task.wait) == "function" then
		task.wait(seconds)
	elseif type(wait) == "function" then
		wait(seconds)
	end
end

_na_boot.getExecutorRequest = function()
	const host = _na_boot.hostEnv
	const candidates = {
		type(request) == "function" and request or nil,
		type(http_request) == "function" and http_request or nil,
		type(syn) == "table" and type(syn.request) == "function" and syn.request or nil,
		type(http) == "table" and type(http.request) == "function" and http.request or nil,
		type(fluxus) == "table" and type(fluxus.request) == "function" and fluxus.request or nil,
		type(host) == "table" and type(rawget(host, "request")) == "function" and rawget(host, "request") or nil,
		type(host) == "table" and type(rawget(host, "http_request")) == "function" and rawget(host, "http_request") or nil,
	}
	for _, fn in candidates do
		if type(fn) == "function" then
			return fn
		end
	end
	return nil
end

_na_boot.makeHttpPayload = function(url, opts)
	opts = type(opts) == "table" and opts or {}
	const payload = {
		Url = url,
		url = url,
		Method = opts.Method or opts.method or "GET",
		method = opts.method or opts.Method or "GET",
		Headers = opts.Headers or opts.headers or {
			Accept = "*/*",
		},
		Timeout = tonumber(opts.Timeout or opts.timeout) or 10,
		FollowRedirects = opts.FollowRedirects ~= false,
		SslVerify = opts.SslVerify == true,
	}
	if opts.Body ~= nil or opts.body ~= nil then
		payload.Body = opts.Body or opts.body
		payload.body = opts.body or opts.Body
	end
	return payload
end

_na_boot.requestUrl = function(url, opts)
	const requestFn = _na_boot.getExecutorRequest()
	if type(requestFn) ~= "function" then
		return false, nil, "executor request unavailable"
	end
	return pcall(requestFn, _na_boot.makeHttpPayload(url, opts))
end

_na_boot.httpGet = function(url, opts)
	opts = type(opts) == "table" and opts or {}
	if type(url) ~= "string" or url == "" then
		error("missing url", 2)
	end
	const maxAttempts = math.clamp(math.floor(tonumber(opts.maxAttempts or opts.retries) or 5), 1, 10)
	const timeout = tonumber(opts.timeout or opts.Timeout) or 10
	local lastErr
	for attempt = 1, maxAttempts do
		local okReq, response = _na_boot.requestUrl(url, { Method = "GET", Timeout = timeout, Headers = opts.Headers or opts.headers })
		if okReq and response then
			const status = _na_boot.getResponseStatus(response)
			const body = _na_boot.getResponseBody(response)
			if type(response) == "string" and response ~= "" then
				return response
			end
			if (status == nil or (status >= 200 and status < 300) or status == 304) and type(body) == "string" and body ~= "" then
				return body
			end
			lastErr = "HTTP "..tostring(status or "unknown")
			if not _na_boot.isRetryableHttp(response, lastErr) then
				break
			end
		else
			lastErr = tostring(response or "request failed")
		end

		local okGet, body = pcall(function()
			if opts.noCache ~= nil then
				return game:HttpGet(url, opts.noCache)
			end
			return game:HttpGet(url)
		end)
		if okGet and type(body) == "string" and body ~= "" then
			return body
		end
		if not _na_boot.isRetryableHttp(nil, body) and not _na_boot.isRetryableHttp(nil, lastErr) then
			lastErr = tostring(body or lastErr or "request failed")
			break
		end
		lastErr = tostring(body or lastErr or "request failed")
		if attempt < maxAttempts then
			const retryAfter = _na_boot.getRetryAfter(response)
			const delay = retryAfter or math.min(8, (0.65 * (2 ^ (attempt - 1))) + (math.random() * 0.35))
			_na_boot.sleepHttp(delay)
		end
	end
	error(lastErr or "HTTP request failed", 2)
end

_na_boot.bootstrapRemoteSources = {}
_na_boot.prefetchBootstrapRemotes = function()
	const targets = {}
	if type(rawget(_na_boot.privateRoot, "serviceResolver")) ~= "table" then
		targets.serviceResolver = "https://ltseverydayyou.github.io/ServiceResolver.luau"
	end
	const uiProtectorBuild = "session_name_cursed_null_v1"
	const cachedProtector = rawget(_na_boot.privateRoot, "uiProtector")
	if type(cachedProtector) ~= "table" or rawget(cachedProtector, "ready") ~= true or rawget(cachedProtector, "build") ~= uiProtectorBuild then
		targets.uiProtector = "https://ltseverydayyou.github.io/UIprotector.luau"
	end

	local pending = 0
	const canParallel = type(task) == "table" and type(task.spawn) == "function" and type(task.wait) == "function"
	for key, url in targets do
		if canParallel then
			pending += 1
			task.spawn(function()
				local ok, source = pcall(_na_boot.httpGet, url, { maxAttempts = 2; timeout = 5; })
				_na_boot.bootstrapRemoteSources[key] = ok and source or false
				pending -= 1
			end)
		else
			local ok, source = pcall(_na_boot.httpGet, url, { maxAttempts = 2; timeout = 5; })
			_na_boot.bootstrapRemoteSources[key] = ok and source or false
		end
	end
	while pending > 0 do
		task.wait()
	end
end
_na_boot.getBootstrapRemoteSource = function(key, url)
	const cached = _na_boot.bootstrapRemoteSources[key]
	if type(cached) == "string" and cached ~= "" then
		return cached
	end
	return _na_boot.httpGet(url, { maxAttempts = 3; timeout = 5; })
end
_na_boot.prefetchBootstrapRemotes()

const __lt = (function()
	const cached = rawget(_na_boot.privateRoot, "serviceResolver");
	if type(cached) == "table" then
		return cached;
	end;
	const loader = loadstring or load;
	if type(loader) ~= "function" then
		error("Service resolver loader unavailable");
	end;
	const resolver = loader(_na_boot.getBootstrapRemoteSource("serviceResolver", "https://ltseverydayyou.github.io/ServiceResolver.luau"), "@ServiceResolver.luau");
	if type(resolver) ~= "function" then
		error("Service resolver failed to compile");
	end;
	const loaded = resolver();
	if type(loaded) ~= "table" then
		error("Service resolver failed to load");
	end;
	_na_boot.privateRoot.serviceResolver = loaded;
	return loaded;
end)();

const __NAUIProtector = (function()
	const uiProtectorBuild = "session_name_cursed_null_v1";
	const cached = rawget(_na_boot.privateRoot, "uiProtector");
	if type(cached) == "table" and rawget(cached, "ready") == true and rawget(cached, "build") == uiProtectorBuild then
		return cached;
	end;
	if type(cached) == "table" then
		pcall(function()
			if type(cached.cleanup) == "function" then
				cached.cleanup();
			end
		end)
		pcall(function()
			if type(cached.restore) == "function" then
				cached.restore();
			end
		end)
	end
	const loader = loadstring or load;
	if type(loader) ~= "function" then
		error("UI protector loader unavailable");
	end;
	const protector = loader(_na_boot.getBootstrapRemoteSource("uiProtector", "https://ltseverydayyou.github.io/UIprotector.luau"), "@UIprotector.luau");
	if type(protector) ~= "function" then
		error("UI protector failed to compile");
	end;
	const loaded = protector();
	if type(loaded) ~= "table" then
		error("UI protector failed to load");
	end;
	_na_boot.privateRoot.uiProtector = loaded;
	return loaded;
end)();

pcall(function()
	for _, target in { _na_env, _na_shared, _na_boot.runtimeEnv } do
		if type(target) == "table" then
			target.__NAServiceResolver = __lt
			target.__NAUIProtector = __NAUIProtector
		end
	end
	_na_boot.privateRoot.serviceResolver = __lt
	_na_boot.privateRoot.uiProtector = __NAUIProtector
end)

NAbegin=tick()
CMDAUTOFILL={}

const NAmanage={}

NAindex = {}
NAmanage._runtimeState = type(_na_env._NARuntimeState) == "table" and _na_env._NARuntimeState or {}
_na_env._NARuntimeState = NAmanage._runtimeState
NAmanage._runtimeState.runSeq = (tonumber(NAmanage._runtimeState.runSeq) or 0) + 1
NAmanage._runtimeState.spawnActive = type(NAmanage._runtimeState.spawnActive) == "table" and NAmanage._runtimeState.spawnActive or {}
NAmanage._runtimeState.waitingThreads = type(NAmanage._runtimeState.waitingThreads) == "table" and NAmanage._runtimeState.waitingThreads or {}
NAmanage._runtimeState.unloadThread = nil
NAmanage._runToken = {}
NAmanage._runtimeState.runToken = NAmanage._runToken
NAmanage._runtimeState.unloading = false
_na_env._NARunToken = NAmanage._runToken

NAjobs = type(_na_env._NAjobs) == "table" and _na_env._NAjobs or {}
_na_env._NAjobs = NAjobs
NAjobs.jobs = NAjobs.jobs or {}
NAjobs._touchState = NAjobs._touchState or {}
NAjobs._frame = NAjobs._frame or 0
NAjobs._claimed = NAjobs._claimed or {}

Lower    = string.lower
Sub      = string.sub
GSub     = string.gsub
Find     = string.find
Match    = string.match
Format   = string.format

Unpack   = table.unpack
Insert   = table.insert
Concat   = table.concat
Discover = table.find

const _naRawTaskSpawn = task.spawn
const _naRawTaskDelay = task.delay
const _naRawTaskWait = task.wait
const _naRawTaskDefer = task.defer
local Spawn, Delay, Wait, Defer

NAmanage._rawTaskSpawn = _naRawTaskSpawn
NAmanage._rawTaskDelay = _naRawTaskDelay
NAmanage._rawTaskWait = _naRawTaskWait
NAmanage._rawTaskDefer = _naRawTaskDefer
NAmanage._runtimeState.spawnActive = setmetatable(NAmanage._runtimeState.spawnActive, { __mode = "k" })
NAmanage._runtimeState.waitingThreads = setmetatable(NAmanage._runtimeState.waitingThreads, { __mode = "k" })

const function runTrackedTask(token, callback, args)
	const running = coroutine.running()
	if rawget(_na_env, "_NARunToken") ~= token or NAmanage._runtimeState.unloading == true then
		NAmanage._runtimeState.spawnActive[running] = nil
		return
	end
	local rawTaskError
	const results = table.pack(xpcall(function()
		return callback(table.unpack(args, 1, args.n))
	end, function(message)
		rawTaskError = tostring(message or "Unknown error")
		if type(debug) == "table" and type(debug.traceback) == "function" then
			local ok, trace = pcall(debug.traceback, rawTaskError, 2)
			if ok and type(trace) == "string" and trace ~= "" then return trace end
		end
		return rawTaskError
	end))
	NAmanage._runtimeState.spawnActive[running] = nil
	if not results[1] and rawget(_na_env, "_NARunToken") == token and NAmanage._runtimeState.unloading ~= true then
		const trace = tostring(results[2] or rawTaskError or "Unknown error")
		if type(NAmanage.NACallerReportExternalError) == "function" then
			pcall(NAmanage.NACallerReportExternalError, rawTaskError or trace, trace, callback, { context = "NA Tracked Task"; severity = "error" })
		else
			__NARootReportError(rawTaskError or trace, trace, "NA Tracked Task", callback, { severity = "error" })
		end
	end
	return table.unpack(results, 2, results.n)
end

Spawn = function(callback, ...)
	if type(callback) ~= "function" then
		return nil
	end
	const token = NAmanage._runToken
	const args = table.pack(...)
	const thread = _naRawTaskSpawn(runTrackedTask, token, callback, args)
	NAmanage._runtimeState.spawnActive[thread] = true
	return thread
end

Delay = function(seconds, callback, ...)
	if type(callback) ~= "function" then
		return nil
	end
	const token = NAmanage._runToken
	const args = table.pack(...)
	const thread = _naRawTaskDelay(tonumber(seconds) or 0, runTrackedTask, token, callback, args)
	NAmanage._runtimeState.spawnActive[thread] = true
	return thread
end

Defer = function(callback, ...)
	if type(callback) ~= "function" then
		return nil
	end
	const token = NAmanage._runToken
	const args = table.pack(...)
	const thread = _naRawTaskDefer(runTrackedTask, token, callback, args)
	NAmanage._runtimeState.spawnActive[thread] = true
	return thread
end

Wait = function(...)
	const token = NAmanage._runToken
	const running = coroutine.running()
	if type(running) == "thread" then
		NAmanage._runtimeState.waitingThreads[running] = true
	end
	const results = table.pack(_naRawTaskWait(...))
	if type(running) == "thread" then
		NAmanage._runtimeState.waitingThreads[running] = nil
	end
	if rawget(_na_env, "_NARunToken") ~= token or NAmanage._runtimeState.unloading == true then
		if running == NAmanage._runtimeState.unloadThread then
			return table.unpack(results, 1, results.n)
		end
		if type(running) == "thread" then
			_naRawTaskDefer(function()
				pcall(task.cancel, running)
			end)
			return coroutine.yield()
		end
		return nil
	end
	return table.unpack(results, 1, results.n)
end

NAmanage.Wrap = function(callback)
	if type(callback) ~= "function" then
		return function() end
	end
	const token = NAmanage._runToken
	const thread = coroutine.create(function(...)
		return callback(...)
	end)
	NAmanage._runtimeState.spawnActive[thread] = true
	return function(...)
		if rawget(_na_env, "_NARunToken") ~= token or NAmanage._runtimeState.unloading == true then
			pcall(task.cancel, thread)
			NAmanage._runtimeState.spawnActive[thread] = nil
			return
		end
		const results = table.pack(coroutine.resume(thread, ...))
		if not results[1] then
			NAmanage._runtimeState.spawnActive[thread] = nil
			error(results[2], 0)
		end
		if coroutine.status(thread) == "dead" then
			NAmanage._runtimeState.spawnActive[thread] = nil
		end
		return table.unpack(results, 2, results.n)
	end
end

NAmanage.MergeMissing = NAmanage.MergeMissing or function(target, source)
	if type(target) ~= "table" or type(source) ~= "table" then
		return target
	end
	for key, value in source do
		if target[key] == nil then
			target[key] = value
		end
	end
	return target
end

NAmanage.IsActiveRun = NAmanage.IsActiveRun or function(token)
	return rawget(_na_env, "_NARunToken") == (token or NAmanage._runToken)
end

NAmanage.isLiveInstance = NAmanage.isLiveInstance or function(inst)
	if typeof(inst) ~= "Instance" then
		return false
	end
	if inst == game then
		return true
	end
	local ok, parent = pcall(function()
		return inst.Parent
	end)
	return ok and parent ~= nil
end

NAmanage.ensureWeakTable = NAmanage.ensureWeakTable or function(tbl, mode)
	mode = type(mode) == "string" and mode or "k"
	if type(tbl) ~= "table" then
		tbl = {}
	end
	const mt = getmetatable(tbl)
	if type(mt) == "table" and mt.__mode == mode then
		return tbl
	end
	const weak = setmetatable({}, { __mode = mode })
	for key, value in tbl do
		weak[key] = value
	end
	return weak
end

NAmanage.ensureWeakKeyTable = NAmanage.ensureWeakKeyTable or function(tbl)
	return NAmanage.ensureWeakTable(tbl, "k")
end

NAmanage.tryDisconnect = NAmanage.tryDisconnect or function(conn)
	if conn and type(conn.Disconnect) == "function" then
		pcall(function()
			conn:Disconnect()
		end)
	end
	return nil
end

NAmanage.isLiveConnection = NAmanage.isLiveConnection or function(conn)
	if conn == nil then
		return false
	end
	if typeof(conn) == "RBXScriptConnection" then
		local ok, connected = pcall(function()
			return conn.Connected
		end)
		return ok and connected ~= false
	end
	const disconnect = conn and conn.Disconnect
	if type(disconnect) ~= "function" then
		return false
	end
	if type(conn) == "table" then
		const connected = rawget(conn, "Connected")
		if connected ~= nil then
			return connected ~= false
		end
	end
	local ok, connectedValue = pcall(function()
		return conn.Connected
	end)
	if ok and connectedValue ~= nil then
		return connectedValue ~= false
	end
	return true
end

NAmanage.ConnectHumanoidDeath = NAmanage.ConnectHumanoidDeath or function(hum, callback, opts)
	if typeof(hum) ~= "Instance" or not hum:IsA("Humanoid") then
		return nil
	end

	const watcher = {
		Connected = true,
		_hum = hum,
		_conns = {},
		_dead = false,
	}

	const function disconnectAll()
		if not watcher.Connected then
			return
		end
		watcher.Connected = false
		for i = 1, #watcher._conns do
			NAmanage.tryDisconnect(watcher._conns[i])
			watcher._conns[i] = nil
		end
	end

	function watcher:Disconnect()
		disconnectAll()
	end

	const function fire()
		if watcher._dead then
			return
		end
		watcher._dead = true
		disconnectAll()
		if type(callback) ~= "function" then
			return
		end
		if opts and opts.defer == false then
			pcall(callback, hum)
			return
		end
		Defer(function()
			pcall(callback, hum)
		end)
	end

	watcher._conns[#watcher._conns + 1] = hum.Died:Connect(function()
		fire()
	end)
	watcher._conns[#watcher._conns + 1] = hum.StateChanged:Connect(function(_, newState)
		if newState == Enum.HumanoidStateType.Dead then
			fire()
		end
	end)
	watcher._conns[#watcher._conns + 1] = hum.AncestryChanged:Connect(function(_, parent)
		if parent == nil then
			disconnectAll()
		end
	end)

	return watcher
end

NAmanage.pruneInstanceKeyMap = NAmanage.pruneInstanceKeyMap or function(map, onRemove)
	if type(map) ~= "table" then
		return
	end
	for key, value in map do
		if typeof(key) == "Instance" and not NAmanage.isLiveInstance(key) then
			if type(onRemove) == "function" then
				pcall(onRemove, value, key)
			end
			map[key] = nil
		end
	end
end

NAmanage.pruneConnectionArray = NAmanage.pruneConnectionArray or function(list, resolver)
	if type(list) ~= "table" then
		return
	end
	local write = 1
	for i = 1, #list do
		const item = list[i]
		const conn = type(resolver) == "function" and resolver(item) or item
		if NAmanage.isLiveConnection(conn) then
			list[write] = item
			write += 1
		end
	end
	for i = write, #list do
		list[i] = nil
	end
end

NAmanage.pruneInstanceArray = NAmanage.pruneInstanceArray or function(list, onRemove)
	if type(list) ~= "table" then
		return
	end
	local write = 1
	for i = 1, #list do
		const item = list[i]
		local keep = true
		if typeof(item) == "Instance" then
			keep = NAmanage.isLiveInstance(item)
		end
		if keep then
			list[write] = item
			write += 1
		else
			if type(onRemove) == "function" then
				pcall(onRemove, item, i)
			end
		end
	end
	for i = write, #list do
		list[i] = nil
	end
end

NAmanage.pruneConnectionValueMap = NAmanage.pruneConnectionValueMap or function(map)
	if type(map) ~= "table" then
		return
	end
	for key, value in map do
		if not NAmanage.isLiveConnection(value) then
			map[key] = nil
		end
	end
end

NAmanage.pruneInstanceValueMap = NAmanage.pruneInstanceValueMap or function(map, cb)
	if type(map) ~= "table" then
		return
	end
	for key, val in map do
		if typeof(val) == "Instance" and not NAmanage.isLiveInstance(val) then
			if type(cb) == "function" then
				pcall(cb, val, key)
			end
			map[key] = nil
		end
	end
end

NAmanage.pruneInstancePairMap = NAmanage.pruneInstancePairMap or function(map, cb)
	if type(map) ~= "table" then
		return
	end
	for key, val in map do
		local dead = false
		if typeof(key) == "Instance" and not NAmanage.isLiveInstance(key) then
			dead = true
		elseif typeof(val) == "Instance" and not NAmanage.isLiveInstance(val) then
			dead = true
		end
		if dead then
			if type(cb) == "function" then
				pcall(cb, val, key)
			end
			map[key] = nil
		end
	end
end

NAmanage.pruneRecordMap = NAmanage.pruneRecordMap or function(map, fields, cb)
	if type(map) ~= "table" then
		return
	end
	fields = type(fields) == "table" and fields or {}
	for key, rec in map do
		local dead = false
		if typeof(key) == "Instance" and not NAmanage.isLiveInstance(key) then
			dead = true
		elseif typeof(rec) == "Instance" and not NAmanage.isLiveInstance(rec) then
			dead = true
		elseif type(rec) == "table" then
			for i = 1, #fields do
				const val = rec[fields[i]]
				if typeof(val) == "Instance" and not NAmanage.isLiveInstance(val) then
					dead = true
					break
				end
			end
			if rec.removed == true then
				dead = true
			end
		end
		if dead then
			if type(cb) == "function" then
				pcall(cb, rec, key)
			end
			map[key] = nil
		end
	end
end

NAmanage.pruneSparseInstanceArray = NAmanage.pruneSparseInstanceArray or function(list, fields, cb)
	if type(list) ~= "table" then
		return
	end
	fields = type(fields) == "table" and fields or {}
	for key, item in list do
		if type(key) == "number" then
			local dead = false
			if typeof(item) == "Instance" then
				dead = not NAmanage.isLiveInstance(item)
			elseif type(item) == "table" then
				if item.removed == true then
					dead = true
				else
					for i = 1, #fields do
						const val = item[fields[i]]
						if typeof(val) == "Instance" and not NAmanage.isLiveInstance(val) then
							dead = true
							break
						end
					end
				end
			end
			if dead then
				if type(cb) == "function" then
					pcall(cb, item, key)
				end
				list[key] = nil
			end
		end
	end
end

NAmanage.pruneRuntimeInstanceState = NAmanage.pruneRuntimeInstanceState or function()
	const state = NAStuff
	if type(state) ~= "table" then
		return
	end
	if type(NAmanage.ensureRuntimeWeakTables) == "function" then
		pcall(NAmanage.ensureRuntimeWeakTables)
	end

	NAmanage.pruneInstanceKeyMap(state._afTracked)
	NAmanage.pruneInstanceKeyMap(state._afOrigCan)
	NAmanage.pruneInstanceKeyMap(state._afpTracked)
	NAmanage.pruneInstanceKeyMap(state._afpOrigCan)
	NAmanage.pruneInstanceKeyMap(state._aaTracked)
	NAmanage.pruneInstanceKeyMap(state._aaOrig)
	NAmanage.pruneInstanceKeyMap(state._godOrig)
	NAmanage.pruneInstanceKeyMap(state._kbMovedParts)
	NAmanage.pruneInstanceKeyMap(state._kbTouchParts)
	NAmanage.pruneInstanceKeyMap(state._kbTouchOriginal)
	NAmanage.pruneInstanceKeyMap(state.partESPGlassOriginal)
	NAmanage.pruneInstanceKeyMap(state.partESPGlassCount)
	NAmanage.pruneInstanceKeyMap(state.partESPLocalTransOriginal)
	NAmanage.pruneInstanceKeyMap(state.partESPLocalTransCount)

	NAmanage.pruneInstanceKeyMap(state._afSignals, function(conn)
		NAmanage.tryDisconnect(conn)
	end)
	NAmanage.pruneInstanceKeyMap(state._afpSignals, function(arr)
		if type(arr) == "table" then
			for i = 1, #arr do
				arr[i] = NAmanage.tryDisconnect(arr[i])
			end
		end
	end)
	NAmanage.pruneInstanceKeyMap(state._aaSignals, function(conn)
		NAmanage.tryDisconnect(conn)
	end)
	NAmanage.pruneInstanceKeyMap(state._godSignals, function(arr)
		if type(arr) == "table" then
			for i = 1, #arr do
				NAmanage.tryDisconnect(arr[i])
				arr[i] = nil
			end
		end
	end)
	NAmanage.pruneInstanceKeyMap(state.bHumCons, function(rec)
		if type(rec) == "table" and type(rec.conns) == "table" then
			for i = 1, #rec.conns do
				rec.conns[i] = NAmanage.tryDisconnect(rec.conns[i])
			end
		end
	end)
	NAmanage.pruneInstanceKeyMap(state.bToolCons, function(rec)
		if type(rec) == "table" and type(rec.conns) == "table" then
			for i = 1, #rec.conns do
				rec.conns[i] = NAmanage.tryDisconnect(rec.conns[i])
			end
		end
	end)
	NAmanage.pruneInstanceKeyMap(state.bSetCons, function(conn)
		NAmanage.tryDisconnect(conn)
	end)
	NAmanage.pruneInstanceKeyMap(state.bHum)
	NAmanage.pruneInstanceKeyMap(state.bTool)
	NAmanage.pruneInstanceKeyMap(state.bSet)

	if type(state.elementOriginalParent) == "table" then
		for inst, parent in state.elementOriginalParent do
			if not NAmanage.isLiveInstance(inst) or (typeof(parent) == "Instance" and not NAmanage.isLiveInstance(parent)) then
				state.elementOriginalParent[inst] = nil
			end
		end
	end

	NAmanage.pruneConnectionArray(state.LastInputConns)
	NAmanage.pruneConnectionArray(state.PreferredInputConns)
	NAmanage.pruneConnectionArray(state.antiAFKStored, function(item)
		return type(item) == "table" and item.conn or nil
	end)
	NAmanage.pruneInstanceArray(state.shownParts)
	NAmanage.pruneInstanceArray(state.tpTools)
	NAmanage.pruneInstanceArray(state.touchESPList)
	NAmanage.pruneInstanceArray(state.proximityESPList)
	NAmanage.pruneInstanceArray(state.clickESPList)
	NAmanage.pruneInstanceArray(state.itemESPList)
	NAmanage.pruneInstanceArray(state.siteESPList)
	NAmanage.pruneInstanceArray(state.vehicleSiteESPList)
	NAmanage.pruneInstanceArray(state.unanchoredESPList)
	NAmanage.pruneInstanceArray(state.collisiontrueESPList)
	NAmanage.pruneInstanceArray(state.collisionfalseESPList)
	NAmanage.pruneInstanceArray(state.propertyESPList)
	NAmanage.pruneInstanceArray(state.ESP_ModelList)
	NAmanage.pruneInstanceArray(state.BlockedRemotes)
	NAmanage.pruneInstanceArray(state.RobloxVersionRows)

	NAmanage.pruneInstanceKeyMap(state.npcCandidates)
	NAmanage.pruneInstanceKeyMap(state.npcESPList)
	NAmanage.pruneInstanceKeyMap(state.itemESPSet)
	NAmanage.pruneInstanceKeyMap(state.itemESPToolMap)
	NAmanage.pruneInstanceKeyMap(state.itemESPPartMap)
	NAmanage.pruneInstanceKeyMap(state.unanchoredESPSet)
	NAmanage.pruneInstanceKeyMap(state.collisiontrueESPSet)
	NAmanage.pruneInstanceKeyMap(state.collisionfalseESPSet)
	NAmanage.pruneInstanceKeyMap(state.propertyESPSet)
	NAmanage.pruneInstanceKeyMap(state.propertyESPMatchCounts)
	if type(state.propertyESPObjectMaps) == "table" then
		for _, objectMap in state.propertyESPObjectMaps do
			NAmanage.pruneInstanceKeyMap(objectMap)
		end
	end
	NAmanage.pruneInstanceKeyMap(state._ncColl)
	if type(state.ChatTranslator) == "table" and type(state.ChatTranslator.messages) == "table" then
		state.ChatTranslator.messages = NAmanage.ensureWeakKeyTable(state.ChatTranslator.messages)
		NAmanage.pruneInstanceKeyMap(state.ChatTranslator.messages)
	end
	NAmanage.pruneInstanceKeyMap(state._messageCopyHooks, function(hook)
		if type(hook) == "table" and type(hook.cleanup) == "function" then
			pcall(hook.cleanup)
		end
	end)
	NAmanage.pruneInstanceKeyMap(state.BlockedRemoteModes)
	NAmanage.pruneInstanceKeyMap(state.BlockedRemoteReturns)
	NAmanage.pruneInstanceKeyMap(state.BlockedEventSaved)
	NAmanage.pruneInstanceKeyMap(state.BlockedInvokeSaved)
	NAmanage.pruneInstanceKeyMap(state.ESP_OcclusionCache)
	NAmanage.pruneInstanceKeyMap(NAmanage._canvasLayoutCache)
	NAmanage.pruneInstanceKeyMap(NAmanage._canvasHeightCache)
	NAmanage.pruneInstanceValueMap(NAmanage._statCache)
	NAmanage.pruneInstanceKeyMap(NAmanage.toolCache)
	NAmanage.pruneInstanceKeyMap(NAmanage.grabBusy)
	NAmanage.pruneInstanceKeyMap(NAmanage.toolGrabCol)
	if type(state.airMomentum) == "table" then
		NAmanage.pruneConnectionValueMap(state.airMomentum.connections)
		if typeof(state.airMomentum.root) == "Instance" and not NAmanage.isLiveInstance(state.airMomentum.root) then
			state.airMomentum.root = nil
		end
		if typeof(state.airMomentum.hum) == "Instance" and not NAmanage.isLiveInstance(state.airMomentum.hum) then
			state.airMomentum.hum = nil
		end
	end
	if type(NAmanage._charAddHub) == "table" then
		NAmanage.pruneInstanceKeyMap(NAmanage._charAddHub.pending)
	end
	if type(NAmanage.pruneChatLogState) == "function" then
		NAmanage.pruneChatLogState()
	end
	if type(NAmanage.pruneAdminChatRainbow) == "function" then
		NAmanage.pruneAdminChatRainbow()
	end
	if state._rbxDevConsoleCopyTarget and not NAmanage.isLiveInstance(state._rbxDevConsoleCopyTarget) then
		if type(NAmanage.cleanupRobloxDevConsoleCopyButtons) == "function" then
			pcall(NAmanage.cleanupRobloxDevConsoleCopyButtons)
		else
			state._rbxDevConsoleCopyTarget = nil
			state._rbxDevConsoleCopyRefresh = nil
			state._rbxDevConsoleCopyCleanup = nil
		end
	end
	if type(NAmanage._descHubs) == "table" and type(NAmanage._descHubDispose) == "function" then
		for root, hub in NAmanage._descHubs do
			if not NAmanage.isLiveInstance(root) or type(hub) ~= "table" or hub.root ~= root or hub.alive == false then
				pcall(NAmanage._descHubDispose, root, hub)
			end
		end
	end
	if type(NAmanage._childHubs) == "table" and type(NAmanage._childHubDispose) == "function" then
		for root, hub in NAmanage._childHubs do
			if not NAmanage.isLiveInstance(root) or type(hub) ~= "table" or hub.root ~= root or hub.alive == false then
				pcall(NAmanage._childHubDispose, root, hub)
			end
		end
	end
	if type(NAmanage._mouseMoveHubs) == "table" and type(NAmanage._mouseMoveHubDispose) == "function" then
		for mouseObj, hub in NAmanage._mouseMoveHubs do
			if mouseObj == nil or type(hub) ~= "table" or hub.mouse ~= mouseObj or hub.alive == false or (tonumber(hub.count) or 0) <= 0 then
				pcall(NAmanage._mouseMoveHubDispose, mouseObj, hub)
			end
		end
	end

	if type(state.CommandLabelPool) == "table" then
		for name, label in state.CommandLabelPool do
			if typeof(label) == "Instance" and not NAmanage.isLiveInstance(label) then
				state.CommandLabelPool[name] = nil
			end
		end
	end

	if type(state.tviewBillboards) == "table" then
		for plr, data in state.tviewBillboards do
			if not NAmanage.isLiveInstance(plr) or type(data) ~= "table" then
				state.tviewBillboards[plr] = nil
			elseif (data.bb and not NAmanage.isLiveInstance(data.bb))
				or (data.head and not NAmanage.isLiveInstance(data.head))
				or (data.char and not NAmanage.isLiveInstance(data.char)) then
				if type(NAmanage.tvDetach) == "function" then
					pcall(NAmanage.tvDetach, plr)
				else
					state.tviewBillboards[plr] = nil
				end
			end
		end
	end

	if type(state.airwalk) == "table" then
		NAmanage.pruneConnectionValueMap(state.airwalk.connections)
		if type(state.airwalk.guis) == "table" then
			for key, gui in state.airwalk.guis do
				if typeof(gui) == "Instance" and not NAmanage.isLiveInstance(gui) then
					state.airwalk.guis[key] = nil
				end
			end
		end
	end
	if type(state._commandKeybindUI) == "table" then
		const root = state._commandKeybindUI.root
		if typeof(root) == "Instance" and not NAmanage.isLiveInstance(root) then
			state._commandKeybindUI = nil
		end
	end

	const lState = _na_env and _na_env._LState
	if type(lState) == "table" then
		for _, key in { "ne", "nf" } do
			const rec = lState[key]
			if type(rec) == "table" and type(rec.cache) == "table" then
				rec.cache = NAmanage.ensureWeakKeyTable(rec.cache)
				NAmanage.pruneInstanceKeyMap(rec.cache)
			end
		end
	end


	if type(NAindex) == "table" and type(NAindex.pruneCaches) == "function" then
		pcall(NAindex.pruneCaches)
	end

	if type(state.partESPEntries) == "table" then
		const staleEntries = {}
		for _, entry in state.partESPEntries do
			if type(entry) == "table" then
				const deadPart = typeof(entry.part) == "Instance" and not NAmanage.isLiveInstance(entry.part)
				const deadKey = typeof(entry.entryKey) == "Instance" and not NAmanage.isLiveInstance(entry.entryKey)
				const deadVisual = typeof(entry.visual) == "Instance" and not NAmanage.isLiveInstance(entry.visual)
				const deadBillboard = typeof(entry.billboard) == "Instance" and not NAmanage.isLiveInstance(entry.billboard)
				if entry.removed or deadPart or deadKey or deadVisual or deadBillboard then
					staleEntries[#staleEntries + 1] = entry
				end
			end
		end
		if #staleEntries > 0 and type(NAmanage.PartESP_UnregisterEntry) == "function" then
			for i = 1, #staleEntries do
				pcall(NAmanage.PartESP_UnregisterEntry, staleEntries[i])
			end
		end
	end

	if type(state.partESPQueue) == "table" then
		NAmanage.pruneSparseInstanceArray(state.partESPQueue, { "part" }, function(item)
			if type(item) == "table" then
				const part = item.part
				if typeof(part) == "Instance" and type(state.partESPQueueMap) == "table" and state.partESPQueueMap[part] == item then
					state.partESPQueueMap[part] = nil
				end
				item.part = nil
				item.guard = nil
			end
		end)
	end
	NAmanage.pruneRecordMap(state.partESPQueueMap, { "part" })
	NAmanage.pruneRecordMap(state.partESPVisualMap, { "part", "visual", "billboard" })

	if type(state.partESPPartMap) == "table" then
		for part, bucket in state.partESPPartMap do
			if not NAmanage.isLiveInstance(part) or type(bucket) ~= "table" then
				state.partESPPartMap[part] = nil
			else
				for entryKey, entry in bucket do
					const badEntry = type(entry) ~= "table" or entry.removed
					const badKey = typeof(entryKey) == "Instance" and not NAmanage.isLiveInstance(entryKey)
					if badEntry or badKey then
						bucket[entryKey] = nil
					end
				end
				if not next(bucket) then
					state.partESPPartMap[part] = nil
				end
			end
		end
	end

	if type(state.folderESPMembers) == "table" then
		for folder, list in state.folderESPMembers do
			if not NAmanage.isLiveInstance(folder) then
				const key = type(state.folderESPKeys) == "table" and state.folderESPKeys[folder] or nil
				if key then
					NAlib.disconnect(key)
					state.folderESPKeys[folder] = nil
				end
				const token = type(state.folderESPScanTokens) == "table" and state.folderESPScanTokens[folder] or nil
				if token and type(NAmanage.CancelTokenCancel) == "function" then
					pcall(NAmanage.CancelTokenCancel, token)
				end
				if type(state.folderESPScanTokens) == "table" then
					state.folderESPScanTokens[folder] = nil
				end
				if type(state.folderESPMemberMaps) == "table" then
					state.folderESPMemberMaps[folder] = nil
				end
				state.folderESPMembers[folder] = nil
			else
				NAmanage.pruneInstanceArray(list)
				const map = type(state.folderESPMemberMaps) == "table" and state.folderESPMemberMaps[folder] or nil
				if type(map) == "table" then
					NAmanage.pruneInstanceKeyMap(map)
				end
			end
		end
	end
	if type(state.folderESPKeys) == "table" then
		NAmanage.pruneInstanceKeyMap(state.folderESPKeys, function(key)
			if key then
				NAlib.disconnect(key)
			end
		end)
	end
	if type(state.folderESPScanTokens) == "table" then
		NAmanage.pruneInstanceKeyMap(state.folderESPScanTokens, function(token)
			if token and type(NAmanage.CancelTokenCancel) == "function" then
				pcall(NAmanage.CancelTokenCancel, token)
			end
		end)
	end
	if type(state.folderESPModes) == "table" then
		NAmanage.pruneInstanceKeyMap(state.folderESPModes)
	end

	if type(state.modelESPMembers) == "table" then
		for model, list in state.modelESPMembers do
			const validModel = NAmanage.isLiveInstance(model) and model:IsA("Model")
			if not validModel then
				const key = type(state.modelESPKeys) == "table" and state.modelESPKeys[model] or nil
				if key then
					NAlib.disconnect(key)
					state.modelESPKeys[model] = nil
				end
				const token = type(state.modelESPScanTokens) == "table" and state.modelESPScanTokens[model] or nil
				if token and type(NAmanage.CancelTokenCancel) == "function" then
					pcall(NAmanage.CancelTokenCancel, token)
				end
				if type(list) == "table" then
					for i = #list, 1, -1 do
						const part = list[i]
						if type(NAmanage.PartESP_QueueRemove) == "function" then
							pcall(NAmanage.PartESP_QueueRemove, part)
						end
						if type(NAmanage.RemoveEspFromPart) == "function" then
							pcall(NAmanage.RemoveEspFromPart, part)
						end
						list[i] = nil
					end
				end
				if type(state.modelESPMemberMaps) == "table" then
					state.modelESPMemberMaps[model] = nil
				end
				if type(state.modelESPMap) == "table" then
					state.modelESPMap[model] = nil
				end
				if type(state.modelESPScanTokens) == "table" then
					state.modelESPScanTokens[model] = nil
				end
				if type(state.modelESPModes) == "table" then
					state.modelESPModes[model] = nil
				end
				state.modelESPMembers[model] = nil
			else
				NAmanage.pruneInstanceArray(list)
				const map = type(state.modelESPMemberMaps) == "table" and state.modelESPMemberMaps[model] or nil
				if type(map) == "table" then
					NAmanage.pruneInstanceKeyMap(map)
				end
			end
		end
	end
	if type(state.modelESPKeys) == "table" then
		NAmanage.pruneInstanceKeyMap(state.modelESPKeys, function(key)
			if key then
				NAlib.disconnect(key)
			end
		end)
	end
	if type(state.modelESPScanTokens) == "table" then
		NAmanage.pruneInstanceKeyMap(state.modelESPScanTokens, function(token)
			if token and type(NAmanage.CancelTokenCancel) == "function" then
				pcall(NAmanage.CancelTokenCancel, token)
			end
		end)
	end
	if type(state.modelESPModes) == "table" then
		NAmanage.pruneInstanceKeyMap(state.modelESPModes)
	end
	if type(state.modelESPModels) == "table" then
		for i = #state.modelESPModels, 1, -1 do
			const model = state.modelESPModels[i]
			if not (NAmanage.isLiveInstance(model) and model:IsA("Model")) then
				if type(NAmanage.ESP_ListRemove) == "function" and type(state.modelESPMap) == "table" then
					pcall(NAmanage.ESP_ListRemove, state.modelESPModels, state.modelESPMap, model)
				else
					table.remove(state.modelESPModels, i)
					if type(state.modelESPMap) == "table" then
						state.modelESPMap[model] = nil
						for idx = i, #state.modelESPModels do
							const current = state.modelESPModels[idx]
							if current ~= nil then
								state.modelESPMap[current] = idx
							end
						end
					end
				end
				if type(state.modelESPModes) == "table" then
					state.modelESPModes[model] = nil
				end
			end
		end
	end

	if type(state.PST) == "table" then
		NAmanage.pruneInstanceKeyMap(state.PST.orig)
		NAmanage.pruneInstanceArray(state.PST.exact)
		NAmanage.pruneInstanceArray(state.PST.partial)
	end

	if type(state.ESP_LocatorArrows) == "table" then
		for key, holder in state.ESP_LocatorArrows do
			if typeof(holder) == "Instance" and not NAmanage.isLiveInstance(holder) then
				state.ESP_LocatorArrows[key] = nil
			end
		end
	end
	if type(state.ESP_PlayerLocatorArrows) == "table" then
		for key, holder in state.ESP_PlayerLocatorArrows do
			if typeof(holder) == "Instance" and not NAmanage.isLiveInstance(holder) then
				state.ESP_PlayerLocatorArrows[key] = nil
			end
		end
	end

	if type(state.activeTeleports) == "table" then
		for key, taskState in state.activeTeleports do
			if type(taskState) ~= "table" or taskState.active ~= true then
				state.activeTeleports[key] = nil
			end
		end
	end

	if type(UserButtonGuiList) == "table" then
		NAmanage.pruneInstanceArray(UserButtonGuiList)
	end
	if type(UserButtonGuiMap) == "table" then
		for id, gui in UserButtonGuiMap do
			if typeof(gui) == "Instance" and not NAmanage.isLiveInstance(gui) then
				UserButtonGuiMap[id] = nil
			end
		end
	end
	if type(UserButtonDropdowns) == "table" then
		for id, drop in UserButtonDropdowns do
			if type(drop) == "table" then
				if drop.conn and not NAmanage.isLiveConnection(drop.conn) then
					drop.conn = nil
				end
				if drop.container and not NAmanage.isLiveInstance(drop.container) then
					drop.container = nil
				end
				if drop.container == nil and drop.conn == nil then
					UserButtonDropdowns[id] = nil
				end
			elseif typeof(drop) == "Instance" and not NAmanage.isLiveInstance(drop) then
				UserButtonDropdowns[id] = nil
			end
		end
	end

	if type(coreGuiProtection) == "table" then
		NAmanage.pruneInstanceKeyMap(coreGuiProtection)
	end
	if type(storedTools) == "table" then
		NAmanage.pruneInstanceArray(storedTools)
	end
	if type(npcCache) == "table" then
		NAmanage.pruneInstanceArray(npcCache)
	end
	if type(NAmanage.pruneInteractionIndex) == "function" then
		pcall(NAmanage.pruneInteractionIndex)
	end
	if type(HumanModCons) == "table" then
		NAmanage.pruneConnectionValueMap(HumanModCons)
	end
	if type(ToolLoopCons) == "table" then
		NAmanage.pruneConnectionValueMap(ToolLoopCons)
	end
	if type(MultiToolCons) == "table" then
		NAmanage.pruneConnectionValueMap(MultiToolCons)
	end
	if type(glueloop) == "table" then
		NAmanage.pruneConnectionValueMap(glueloop)
	end
	if type(glueBACKER) == "table" then
		NAmanage.pruneConnectionValueMap(glueBACKER)
	end

	if type(NAgui) == "table" then
		NAmanage.pruneInstanceKeyMap(NAgui._resizeCleanup, function(cleanup)
			if type(cleanup) == "function" then
				pcall(cleanup)
			end
		end)
		if type(NAgui._toggleRegistry) == "table" then
			for key, entry in NAgui._toggleRegistry do
				const button = type(entry) == "table" and entry.button or nil
				if typeof(button) == "Instance" and not NAmanage.isLiveInstance(button) then
					NAgui._toggleRegistry[key] = nil
				end
			end
		end
		if type(NAgui._colorPickerRegistry) == "table" then
			for key, entry in NAgui._colorPickerRegistry do
				const picker = type(entry) == "table" and entry.picker or nil
				if typeof(picker) == "Instance" and not NAmanage.isLiveInstance(picker) then
					NAgui._colorPickerRegistry[key] = nil
				end
			end
		end
		if type(NAgui._inputRegistry) == "table" then
			for key, entry in NAgui._inputRegistry do
				const input = type(entry) == "table" and entry.input or nil
				if typeof(input) == "Instance" and not NAmanage.isLiveInstance(input) then
					NAgui._inputRegistry[key] = nil
				end
			end
		end
		if type(NAgui._sliderRegistry) == "table" then
			for key, entry in NAgui._sliderRegistry do
				const slider = type(entry) == "table" and entry.slider or nil
				if typeof(slider) == "Instance" and not NAmanage.isLiveInstance(slider) then
					NAgui._sliderRegistry[key] = nil
				end
			end
		end
		if type(NAgui._dropdownRegistry) == "table" then
			for key, entry in NAgui._dropdownRegistry do
				local dropdown = type(entry) == "table" and entry.dropdown or nil
				if dropdown == nil and type(entry) == "table" and type(entry.api) == "table" then
					dropdown = entry.api.Instance
				end
				if typeof(dropdown) == "Instance" and not NAmanage.isLiveInstance(dropdown) then
					NAgui._dropdownRegistry[key] = nil
				end
			end
		end
		if type(NAgui._keybindRegistry) == "table" then
			for key, entry in NAgui._keybindRegistry do
				const keybind = type(entry) == "table" and (entry.keybind or entry.button) or nil
				if typeof(keybind) == "Instance" and not NAmanage.isLiveInstance(keybind) then
					NAgui._keybindRegistry[key] = nil
				end
			end
		end
	end

	if type(TopBarApp) == "table" and type(TopBarApp.childButtons) == "table" then
		NAmanage.pruneInstanceKeyMap(TopBarApp.childButtons)
	end
end

NAmanage._gmx31 = NAmanage._gmx31 or function(bytes, seed)
	const out = {}
	seed = tonumber(seed) or 0
	for i = 1, #bytes do
		const key = ((seed + (i * 3)) % 11) + 17
		out[i] = string.char((tonumber(bytes[i]) or 0) - key)
	end
	return Concat(out)
end
NAmanage._guardTokens = NAmanage._guardTokens or {
	name = NAmanage._gmx31({
		142, 132, 125, 133, 129, 100, 122, 121, 135, 97, 87, 136, 131, 135, 134, 135, 119, 137, 128, 132, 129, 125, 130, 117, 127, 88, 108, 107, 104, 107, 112, 86, 97, 108, 82, 103, 106
	}, 5);
	value = NAmanage._gmx31({
		119, 139, 138, 135, 106, 111, 85, 96, 101, 116, 87, 95, 92, 92
	}, 2);
}
NAmanage._routeGateText = NAmanage._routeGateText or NAmanage._gmx31({
	146, 130, 131, 137, 49, 139, 134, 135, 130, 53, 69, 59, 127, 133, 136, 124, 135, 55, 134, 123, 128, 125, 59, 140, 133, 142, 49, 117, 137, 127, 50, 119, 132, 124, 118, 129, 133, 122, 135, 139, 127, 118, 53, 106, 112, 91, 54, 107, 96, 92, 55, 84, 78
}, 7)
NAmanage._routeGateExit = NAmanage._routeGateExit or function()
	const value = NAmanage._routeGateText
	if type(value) == "string" and #value == 53 then
		local total = 0
		for i = 1, #value do
			total = total + (i * (string.byte(value, i) or 0))
		end
		if total == 119325 then
			return value
		end
	end
	return nil
end
NAmanage._routeGateBlob = NAmanage._routeGateBlob or NAmanage._gmx31({
	54, 89, 86, 78, 131, 107, 149, 88, 125, 74, 112, 106, 145, 107, 139, 109, 120, 87, 141, 100, 78, 99, 128, 107, 148, 129, 95, 106, 86, 126, 144, 59, 119, 97, 62, 132, 80
}, 9)

NAmanage._la0 = NAmanage._la0 or { 122, 135, 136, 133, 137, 81, 64, 65, 133, 117, 140, 68 }
NAmanage._cf0 = NAmanage._cf0 or { 86, 133, 139, 118 }
NAmanage._lx2 = NAmanage._lx2 or { 101, 136, 132, 133, 133, 137, 133, 64, 127, 137, 118, 139 }

local Notify = nil
local Window = nil
local Popup  = nil

const NA_TABS = {
	TAB_ALL = "All";
	TAB_GENERAL = "General";
	TAB_INTEGRATIONS = "Integrations";
	TAB_INTERFACE = "Interface";
	TAB_FFLAGS = "FFlags";
	TAB_ENGINE_SETTINGS = "Engine Settings";
	TAB_AUTOMATION = "Automation";
	TAB_MANAGEMENT = "Management";
	TAB_SAVE_INSTANCE = "Save Instance";
	TAB_USER_BUTTONS = "User Buttons";
	TAB_LOGGING = "Logging";
	TAB_ESP = "ESP";
	TAB_CHAT = "Chat";
	TAB_CHARACTER = "Character";
	TAB_KEYBINDS = "Keybinds";
	TAB_BASIC_INFO = "Basic Info";
	TAB_ROBLOX_DATA = "Roblox Data";
	TAB_CONTRIBUTORS = "Contributors";
	TAB_ADMIN_INFO = "NA Info";
}

local NAStuff = {
	prefixCheck = ";";
	Notification = nil;
	CmdBar2 = (NAStuff and NAStuff.CmdBar2) or {
		defaultWidth = 340;
		defaultHeight = 78;
		minWidth = 200;
		maxWidth = 500;
		minHeight = 70;
		maxHeight = 160;
		topHeight = 26;
		bodyOffsetY = 34;
		bodyBottomPadding = 8;
		bodyMinHeight = 24;
	};
	NAICONMAIN = nil;
	NASCREENGUI = nil; --Getmodel("rbxassetid://140418556029404")
	NAjson = nil;
	nuhuhNotifs = true;
	dmNotificationsEnabled = true;
	inviteLink = { 122, 135, 136, 133, 137, 81, 64, 65, 119, 125, 136, 121, 134, 131, 118, 65, 123, 124, 69, 145, 139, 124, 108, 124, 137, 99, 94, 87, 86 };
	docsLink = { 122, 135, 136, 133, 137, 81, 64, 65, 127, 136, 136, 123, 141, 118, 132, 140, 120, 118, 143, 144, 128, 135, 65, 123, 126, 138, 127, 134, 116, 65, 125, 132, 69, 101, 82, 63, 119, 131, 120, 137 };
	supportLink = { 122, 135, 136, 133, 137, 81, 64, 65, 126, 131, 66, 124, 128, 63, 117, 130, 129, 68, 130, 139, 132, 119, 137, 121, 135, 143, 123, 114, 139, 140, 131, 138 };
	officialRepoLink = { 122, 135, 136, 133, 137, 81, 64, 65, 122, 125, 137, 126, 140, 115, 64, 118, 131, 130, 69, 131, 133, 133, 120, 138, 122, 136, 144, 117, 115, 140, 141, 132, 139, 70, 95, 115, 128, 121, 129, 123, 138, 132, 63, 84, 120, 130, 127, 133 };
	_lb0 = { 127, 132, 135, 126, 142, 115, 137, 138, 127, 132, 120, 135, 137, 135, 123, 135, 133, 66 };
	_cf1 = { 89, 136, 125, 98, 119, 133, 122, 130 };
	_lx1 = { 84, 129, 123 };
	KeybindConnection = nil;
	StreamerModeEnabled = false;
	StreamerModeText = "User";
	StreamerModeGray = Color3.fromRGB(127, 127, 127);
	StreamerModeImage = "rbxasset://textures/LightThemeLoadingCircle.png";
	StreamerModeState = {
		cache = {};
		nameTokens = {};
		token = nil;
		restoreToken = nil;
		applied = false;
		scrubBusy = false;
	};
	AutoExecEnabled = true;
	UserButtonsAutoLoad = true;
	StartupInitializersReady = false;
	FriendRequestAutoDismiss = false;
	ConnectionsToFriends = false;
	UnsafeFunctionsDisabled = false;
	UnsafeFunctionState = {
		originals = {};
	};
	CmdBar2AutoRun = false;
	CmdInputSafeMode = true;
	HideCmdAutofill = false;
	LegacyCommandUI = false;
	LegacyHorizontalSettingsTabs = false;
	SettingsSidebarCompactMode = false;
	CmdIntegrationAutoRun = false;
	CmdIntegrationLoaded = false;
	CmdIntegrationLastSource = nil;
	CmdIntegrationRoutingMode = "NA First";
	CmdIntegrationExposeGateway = true;
	CmdIntegrationUseNotifications = true;
	CmdIntegrationMirrorNotifications = false;
	IYIntegrationAutoRun = false;
	IYIntegrationLoaded = false;
	IYIntegrationLastSource = nil;
	IYIntegrationRoutingMode = "NA First";
	IYIntegrationExposeGateway = true;
	IYIntegrationMirrorNotifications = false;
	cmdAutofillLoading = false;
	cmdAutofillLoadRequested = false;
	uiBootHidden = false;
	HideStartup = false;
	keepCmdFocus = false;
	cmdInputAtInit = nil;
	tweenSpeed = 1;
	tpDelay = 0.2;
	AutoInteractDefaultInterval = 0.1;
	AutoFireRemoteDefaultInterval = 0.1;
	AutoInteractMethod = "PostSimulation";
	AutoFireRemoteMethod = "PostSimulation";
	ClickTouchMaxDistance = 1024;
	ClickTouchScreenRadius = 18;
	ClickTouchBlockedByCollide = false;
	ClickTouchIgnoreNonCollideBlockers = true;
	ClickTouchInvisibleFallback = true;
	ClickTouchAlwaysOnTop = false;
	FreecamSpeed = 5;
	MobileFlyAutoEnableOnRun = true;
	MobileCamSensitivity = 1;
	MobileCamSensEnabled = false;
	CFlyVisualizerOn = true;
	OffVisOn = true;
	OffVisAcc = true;
	OffVisFTr = 0.82;
	OffVisOTr = 0.15;
	BlockedRemotes = {};
	touchESPList = {};
	proximityESPList = {};
	clickESPList = {};
	itemESPList = {};
	itemESPSet = {};
	itemESPToolMap = {};
	itemESPPartMap = {};
	itemESPEnabled = false;
	siteESPList = {};
	vehicleSiteESPList = {};
	unanchoredESPList = {};
	unanchoredESPSet = {};
	collisiontrueESPList = {};
	collisiontrueESPSet = {};
	collisionfalseESPList = {};
	collisionfalseESPSet = {};
	propertyESPList = {};
	propertyESPSet = {};
	propertyESPMatchCounts = {};
	propertyESPObjectMaps = {};
	propertyESPQueries = {};
	espTriggers = {};
	espNameLists = { exact = {}, partial = {} };
	espNameTriggers = {};
	nameESPPartLists = { exact = {}, partial = {} };
	nameESPPartMaps = {
		exact = {},
		partial = {},
	};
	ESP_RenderMode = "Highlight";
	ESP_PartRenderMode = "BoxHandleAdornment";
	ESP_Transparency = 0.7;
	ESP_PartTransparency = 0.45;
	ESP_LabelTextSize = 12;
	ESP_LabelTextScaled = false;
	ESP_LabelStrokeTransparency = 0.5;
	ESP_DrawingTextOutline = true;
	ESP_DrawingTextCentered = true;
	ESP_DrawingTextTransparency = 0;
	ESP_DrawingTextFont = "UI";
	ESP_DrawingBoxStyle = "Square";
	ESP_DrawingBoxThickness = 1;
	ESP_DrawingBoxOutline = true;
	ESP_DrawingBoxOutlineThickness = 3;
	ESP_DrawingFilledBoxes = false;
	ESP_DrawingCornerScale = 0.25;
	ESP_DrawingPartBoxStyle = "Square";
	ESP_DrawingPartTextOutline = true;
	ESP_DrawingPartTextCentered = true;
	ESP_DrawingPartTextTransparency = 0;
	ESP_DrawingPartBoxThickness = 1;
	ESP_DrawingPartBoxOutline = true;
	ESP_DrawingPartBoxOutlineThickness = 3;
	ESP_DrawingPartFilledBoxes = false;
	ESP_DrawingPartCornerScale = 0.25;
	ESP_DrawingPartQueuePerStep = 64;
	ESP_DrawingMaxPerStep = 64;
	ESP_PartUpdatePerStep = 48;
	ESP_PartMaxActive = 450;
	ESP_PartSweepInterval = 4;
	ESP_PartNameSweepInterval = 18;
	ESP_NameWatchPerStep = 96;
	ESP_DrawingTracerEnabled = false;
	ESP_DrawingTracerOrigin = "Bottom";
	ESP_DrawingTracerTarget = "Bottom";
	ESP_DrawingTracerThickness = 1;
	ESP_DrawingTracerOutline = true;
	ESP_OcclusionEnabled = false;
	ESP_OcclusionIncludePlayers = false;
	ESP_OcclusionIncludeNPCs = false;
	ESP_OcclusionIncludeParts = false;
	ESP_OcclusionHideLabels = false;
	ESP_OcclusionHidePartLabels = false;
	ESP_OcclusionHideTracers = false;
	ESP_OcclusionDimBoxes = false;
	ESP_OcclusionDimLabels = false;
	ESP_OcclusionIgnoreTransparent = false;
	ESP_OcclusionTransparentThreshold = 0.85;
	ESP_OcclusionIgnoreNonCollidable = false;
	ESP_OcclusionIgnoreSameModel = false;
	ESP_OcclusionHitProbeLimit = 4;
	ESP_OcclusionDimAmount = 0.55;
	ESP_OcclusionUpdateInterval = 0.25;
	ESP_OcclusionMaxPerStep = 12;
	ESP_OcclusionMaxDistance = 1500;
	ESP_OcclusionColor = Color3.fromRGB(130, 130, 130);
	ESP_OcclusionCache = setmetatable({}, { __mode = "k" });
	ESP_UseCustomColor = false;
	ESP_CustomColor = Color3.new(1, 1, 1);
	ESP_OutlineTransparency = 0;
	ESP_IgnoreTeam = false;
	ESP_TargetTeam = "";
	ESP_PlayerTargetMode = "all";
	ESP_ShowPartText = true;
	ESP_ShowPartDistance = false;
	ESP_PartColor_Name = Color3.fromRGB(255, 255, 255);
	ESP_PartColor_Folder = Color3.fromRGB(255, 220, 0);
	ESP_PartColor_Model = Color3.fromRGB(0, 200, 255);
	ESP_PartColor_Touch = Color3.fromRGB(255, 0, 0);
	ESP_PartColor_Proximity = Color3.fromRGB(0, 0, 255);
	ESP_PartColor_Click = Color3.fromRGB(255, 165, 0);
	ESP_PartColor_Item = Color3.fromRGB(90, 255, 135);
	ESP_PartColor_Seat = Color3.fromRGB(0, 255, 0);
	ESP_PartColor_VehicleSeat = Color3.fromRGB(255, 0, 255);
	ESP_PartColor_Unanchored = Color3.fromRGB(255, 220, 0);
	ESP_PartColor_CollisionTrue = Color3.fromRGB(0, 200, 255);
	ESP_PartColor_CollisionFalse = Color3.fromRGB(255, 120, 120);
	ESP_PartColor_Property = Color3.fromRGB(190, 120, 255);
	ESP_LocatorEnabled = false;
	ESP_LocatorSize = 26;
	ESP_LocatorShowText = false;
	ESP_LocatorTextSize = 14;
	ESP_PlayerLocatorEnabled = false;
	ESP_PlayerLocatorSize = 26;
	ESP_PlayerLocatorShowText = false;
	ESP_PlayerLocatorTextSize = 14;
	WaypointESP_ShowBox = false;
	WaypointESP_ShowName = true;
	WaypointESP_ShowDistance = true;
	WaypointESP_ShowIcon = true;
	WaypointESP_ShowHighlight = true;
	WaypointESP_MaxDistance = 100000;
	WaypointESP_IconSize = 42;
	WaypointESP_TextSize = 18;
	WaypointESP_Color = Color3.fromRGB(75, 155, 255);
	WaypointPath_ShowNodes = true;
	WaypointPath_ShowText = false;
	WaypointPath_LoopMode = "walk";
	WaypointPath_TweenSpeed = 24;
	WaypointPath_TeleportDelay = 0.25;
	ESP_LastExactPart = "";
	ESP_LastPartialPart = "";
	ESP_LastShapeESP = "Block";
	ESP_LastPropertyESP = "";
	ESP_LastFolderName = "";
	ESP_LastModelName = "";
	ESP_LocatorGui = nil;
	ESP_LocatorArrows = {};
	ESP_PlayerLocatorGui = nil;
	ESP_PlayerLocatorArrows = {};
	ESP_ModelList = {};
	ESP_ModelIndex = 1;
	ESP_MaxPerStep = 24;
	ESP_ScanBatchSize = 160;
	ESP_ScanDelay = 0;
	ESP_RescanPerStep = 90;
	ESP_FolderMode = "parts";
	ESP_ModelMode = "parts";
	NPC_ESP_RenderMode = "Highlight";
	NPC_ESP_MaxDist = 400;
	NPC_ESP_MaxCount = 200;
	NPC_ESP_ShowLabels = true;
	NPC_ESP_LabelMaxDistance = 600;
	partESPColors = {};
	partESPGlassOriginal = {};
	partESPGlassCount = {};
	partESPEntries = {};
	partESPUpdateCursor = nil;
	partESPVisualMap = {};
	partESPQueueMap = {};
	partESPQueue = {};
	partESPQueueHead = 1;
	partESPQueueTail = 0;
	espScanTokens = {};
	espSweepCursor = {};
	nameESPExclusions = { exact = {}, partial = {} };
	CrosshairGap = 2;
	CrosshairShowCenter = true;
	TopbarGlassTransparency = 0.12;
	TopbarStrokeTransparency = 0.15;
	TopbarPanelTransparency = 0.1;
	TopbarButtonTransparency = 0.18;
	TopbarButtonShape = "Circle";
	RobloxTopbarEditorEnabled = false;
	RobloxTopbarLayout = "Controls Unified";
	RobloxTopbarMorePosition = "Original";
	RobloxTopbarBackgroundColor = Color3.fromRGB(18, 18, 21);
	RobloxTopbarBackgroundTransparency = 0.08;
	RobloxTopbarCornerRadius = 22;
	RobloxTopbarSeparateGap = 4;
	SideSwipeWidth = 80;
	SideSwipePanelHeight = 0;
	SideSwipeHandleWidth = 0;
	SideSwipeHandleHeight = 0;
	SideSwipeHandleVerticalPosition = 50;
	SideSwipeSwipeThreshold = 28;
	SideSwipeButtonHeight = 48;
	SideSwipeButtonSpacing = 8;
	SideSwipeHandleTransparency = 0.72;
	SideSwipePanelTransparency = 0.35;
	SideSwipeButtonTransparency = 0.16;
	SideSwipeScrollBarThickness = 4;
	Integrations = {
		webhook = {
			url = "";
			urls = { main = "" };
			enableJoinLeave = false;
			enableChat = false;
			enableCommands = false;
			minInterval = 2;
			lastSent = 0;
			lastSentByKind = {};
			mainMessage = "";
			rawPayload = [[{"content":"Hello from Nameless Admin"}]];
			options = {
				enabled = true;
				username = "Nameless Admin";
				avatarUrl = "";
				useEmbeds = true;
				titlePrefix = "Nameless Admin";
				footerText = "Nameless Admin";
				thumbnailUrl = "";
				imageUrl = "";
				includeTimestamp = true;
				includeServerInfo = true;
				blockMentions = true;
				silent = false;
				tts = false;
				separateCooldowns = true;
				colors = {
					join = "57F287";
					leave = "ED4245";
					chat = "5865F2";
					command = "FEE75C";
					main = "7C3AED";
					test = "EB459E";
				};
			};
			templates = {
				join = "**{player}** joined the server.";
				leave = "**{player}** left the server.";
				chat = "**{player}:** {message}";
				command = "**{player}** ran `{command}`";
			};
			stats = { sent = 0; failed = 0; lastStatus = nil; lastError = ""; lastSentAt = 0; };
		};
		health = {
			endpoints = {};
		};
		notes = {
			last = "";
		};
		rpc = {
			useCustom = false;
			details = "";
			state = "";
		};
	};
	NIL_SENTINEL = {};
	RemoteBlockMode = "fakeok";
	RemoteFakeReturn = true;
	BlockedEventSaved = {};
	BlockedInvokeSaved = {};
	BlockedRemoteModes = {};
	BlockedRemoteReturns = {};
	BlockedSignals = {};
	RemoteFakeReturn = true;
	AntiKickMode = "fakeok";
	AntiKickHooked = false;
	AntiKickOrig = {namecall=nil,index=nil,newindex=nil,kicks={}};
	AntiTeleportMode = "fakeok";
	AntiTeleportHooked = false;
	AntiTeleportOrig = {namecall=nil,index=nil,newindex=nil,funcs={}};
	SYNC_TAG = "ANIM_SYNC";
	CORE_FOLDERS = {idle=true,walk=true,run=true,jump=true,fall=true,climb=true,swim=true,swimidle=true,toolnone=true,toolslash=true,toollunge=true};
	SavedDefaultMap = nil;
	Sync_AnimatePrevDisabled = nil;
	Sync_Stop = nil;
	MIMIC_TAG = "MIMIC_SYNC";
	Mimic_AnimatePrevDisabled = nil;
	Mimic_Stop = nil;
	mimic_uid = 0;
	ChatSettings = {
		customEnabled = false;
		coreGuiChat = true;
		coreGuiChatLoop = false;
		window = {
			enabled = true;
			font = "rbxasset://fonts/families/BuilderSans.json";
			widthScale = 1;
			heightScale = 1;
			horizontalAlignment = "Left";
			verticalAlignment = "Top";
			textSize = 16;
			textColor = {235,235,235};
			strokeColor = {0,0,0};
			strokeTransparency = 0.5;
			textTransparency = 0;
			backgroundColor = {25,27,29};
			backgroundTransparency = 0.2;
		};
		tabs = {
			enabled = false;
			font = "rbxasset://fonts/families/BuilderSans.json";
			textSize = 18;
			backgroundColor = {25,27,29};
			backgroundTransparency = 0;
			hoverBackgroundColor = {41,44,48};
			textTransparency = 0;
			textColor = {255,255,255};
			selectedTextColor = {170,255,170};
			unselectedTextColor = {200,200,200};
			strokeColor = {0,0,0};
			strokeTransparency = 0.5;
		};
		input = {
			enabled = true;
			autocomplete = true;
			font = "rbxasset://fonts/families/BuilderSans.json";
			keyCode = "Slash";
			targetChannel = "";
			textSize = 16;
			textColor = {255,255,255};
			strokeColor = {0,0,0};
			strokeTransparency = 0.5;
			placeholderColor = {178,178,178};
			backgroundColor = {25,27,29};
			backgroundTransparency = 0.2;
			targetGeneral = false;
		};
		bubbles = {
			enabled = true; -- ENABLED IT SINCE YOU CAN'T STOP CRYING ABOUT IT
			font = "";
			adorneeName = "HumanoidRootPart";
			localPlayerStudsOffset = {0,0,0};
			maxDistance = 100;
			minimizeDistance = 20;
			verticalStudsOffset = 0;
			textSize = 14;
			textColor = {255,255,255};
			textTransparency = 0;
			spacing = 4;
			backgroundColor = {25,27,29};
			backgroundTransparency = 0.1;
			maxBubbles = 3;
			bubbleDuration = 15;
			tailVisible = true;
		};
	};
	ChatSettingsTemplate = nil;
	ChatSettingsDefaults = nil;
	ChatCustomizationActive = nil;
	ChatSettingsCustomBackup = nil;
	IconInvisible = false;
	IconLocked = false;
	_prefetchedRemotes = {};
	AutoExecBlockedCommands = {
		exit = true;
		rejoin = true;
		rj = true;
		serverhop = true;
		shop = true;
		smallserverhop = true;
		sshop = true;
		pingserverhop = true;
		pshop = true;
		oldserverhop = true;
		oldhop = true;
		newserverhop = true;
		newhop = true;
		versionhop = true;
		vhop = true;
		oldversionhop = true;
		ovhop = true;
		regionhop = true;
		rhop = true;
	};
	NASettingsSchema = nil;
	NASettingsData = nil;
	elementOriginalParent = {};
	_lastCommand = nil;
	_prevCommand = nil;
	_removeAdsLoop = nil;
	resizeVerticalAsset = nil;
	resizeHorizontalAsset = nil;
	resizeDiagonal1Asset = nil;
	resizeDiagonal2Asset = nil;
	defaultCmdClear = nil;
	autofillSelecting = false;
	cmdFocusGuardUntil = 0;
	autofillRefocusGuard = 0;
	cmdBarSelected = false;
	cmdAutofillClickable = false;
	cmdSearchSuspendUntil = 0;
}

do
	if type(_na_env._NAStuff) == "table" and _na_env._NAStuff ~= NAStuff then
		NAmanage.MergeMissing(_na_env._NAStuff, NAStuff)
		NAStuff = _na_env._NAStuff
	else
		_na_env._NAStuff = NAStuff
	end
end

NAmanage.ensureRuntimeWeakTables = NAmanage.ensureRuntimeWeakTables or function()
	if type(NAStuff) ~= "table" or type(NAmanage.ensureWeakTable) ~= "function" then
		return
	end

	if type(NAStuff.StreamerModeState) == "table" then
		NAStuff.StreamerModeState.cache = NAmanage.ensureWeakTable(NAStuff.StreamerModeState.cache, "k")
	end

	for _, field in {
		"unanchoredESPSet",
		"collisiontrueESPSet",
		"collisionfalseESPSet",
		"propertyESPSet",
		"propertyESPMatchCounts",
		"ESP_OcclusionCache",
		"partESPGlassOriginal",
		"partESPGlassCount",
		"partESPLocalTransOriginal",
		"partESPLocalTransCount",
		"partESPEntries",
		"partESPQueueMap",
		"partESPVisualMap",
		"partESPPartMap",
		"folderESPMembers",
		"folderESPKeys",
		"folderESPScanTokens",
		"folderESPModes",
		"folderESPMemberMaps",
		"modelESPMembers",
		"modelESPKeys",
		"modelESPScanTokens",
		"modelESPModes",
		"modelESPMemberMaps",
		"modelESPMap",
		"BlockedEventSaved",
		"BlockedInvokeSaved",
		"BlockedRemoteModes",
		"BlockedRemoteReturns",
		"BlockedSignals",
	} do
		NAStuff[field] = NAmanage.ensureWeakTable(NAStuff[field], "k")
	end

	NAStuff.elementOriginalParent = NAmanage.ensureWeakTable(NAStuff.elementOriginalParent, "kv")
	NAStuff.genericESPListMeta = NAmanage.ensureWeakTable(NAStuff.genericESPListMeta, "k")
end

NAmanage.ensureRuntimeWeakTables()

NAmanage._lc0 = NAmanage._lc0 or { 117, 130, 129, 68, 130, 139, 132, 119, 137, 121, 135, 143, 123, 114, 139, 140, 131, 138, 69 }
NAmanage._ld0 = NAmanage._ld0 or { 143, 135, 138, 141, 144, 136, 139, 72, 131, 121, 125, 141, 65 }

NAmanage._sourceGlyph = NAmanage._sourceGlyph or function(value)
	if type(value) == "function" then
		local ok, result = pcall(value)
		if ok then
			return tostring(result or "")
		end
		return ""
	end
	if type(value) == "string" then
		return value
	end
	if type(value) ~= "table" then
		return ""
	end

	const chars = {}
	for i = 1, #value do
		const num = tonumber(value[i])
		if not num then
			return ""
		end
		chars[i] = string.char(num - ((i % 7) + 17))
	end
	return Concat(chars)
end

NAmanage._linkGlyph = NAmanage._linkGlyph or function(parts)
	if type(parts) ~= "table" then
		return ""
	end
	const out = {}
	for i = 1, #parts do
		const piece = parts[i]
		if type(piece) == "table" and rawget(piece, "m") == true then
			out[i] = NAmanage._gmx31(rawget(piece, "b"), rawget(piece, "s"))
		else
			out[i] = NAmanage._sourceGlyph(piece)
		end
	end
	return Concat(out)
end

local opt = {}

const LoadstringCommandAliases = {
	loadstring = true;
	ls = true;
	lstring = true;
	loads = true;
	execute = true;
};

NAmanage._le0 = NAmanage._le0 or { 122, 120, 117, 121, 137, 70, 126, 115, 124, 130, 68 }
NAmanage._cf2 = NAmanage._cf2 or { 138, 132, 124, 135, 127, 136, 127, 66, 131, 143, 115, 138 }
NAmanage._lx0 = NAmanage._lx0 or { 126, 139, 136, 122 }

NAmanage.NA_getServiceRef = function(name)
	if type(cloneref) == "function" and type(__lt.cs) == "function" then
		return __lt.cs(name, cloneref)
	end
	return __lt.gs(name)
end
NAmanage.NA_getServiceRaw = NAmanage.NA_getServiceRaw or function(name)
	return __lt.gs(name)
end

const NA_SRV = setmetatable({}, {
	__index = function(self, name)
		local ok, svc = pcall(NAmanage.NA_getServiceRef, name)
		if ok and svc then
			rawset(self, name, svc)
			return svc
		end
	end
})

const NA_SRV_RAW = setmetatable({}, {
	__index = function(self, name)
		local ok, svc = pcall(NAmanage.NA_getServiceRaw, name)
		if ok and svc then
			rawset(self, name, svc)
			return svc
		end
	end
})

function SafeGetService(name, useCloneRef)
	if useCloneRef == false then
		return NA_SRV_RAW[name]
	end
	const cached = rawget(NA_SRV, name)
	if cached ~= nil then
		return cached
	end
	return NA_SRV[name]
end

const Services = {
	Workspace = SafeGetService("Workspace");
	HttpService = SafeGetService("HttpService");
	Players = SafeGetService("Players");
	UserService = SafeGetService("UserService");
	UserInputService = SafeGetService("UserInputService");
	TweenService = SafeGetService("TweenService");
	RunService = SafeGetService("RunService");
	ContextActionService = SafeGetService("ContextActionService");
	TeleportService = SafeGetService("TeleportService");
	ExperienceService = SafeGetService("ExperienceService");
	Lighting = SafeGetService("Lighting");
	ReplicatedStorage = SafeGetService("ReplicatedStorage");
	CoreGui = SafeGetService("CoreGui");
	SoundService = SafeGetService("SoundService");
	TextChatService = SafeGetService("TextChatService");
	TextService = SafeGetService("TextService");
	StarterGui = SafeGetService("StarterGui");
	ContentProvider = SafeGetService("ContentProvider");
	LocalizationService = SafeGetService("LocalizationService");
	MarketplaceService = SafeGetService("MarketplaceService");
	GuiService = SafeGetService("GuiService");
	Stats = SafeGetService("Stats");
	LogService = SafeGetService("LogService");
}

NAmanage.SafeCloneRef = NAmanage.SafeCloneRef or function(value)
	if value == nil then
		return nil
	end
	if type(__lt.cv) == "function" then
		local ok, cloned = pcall(__lt.cv, value)
		if ok and cloned ~= nil then
			return cloned
		end
	end
	return value
end

NAmanage.GetMouse = NAmanage.GetMouse or function(plr)
	if not plr then
		const ps = SafeGetService and SafeGetService("Players") or nil
		plr = ps and ps.LocalPlayer or nil
	end
	if not plr then
		return nil
	end
	local ok, mouse = pcall(function()
		return plr:GetMouse()
	end)
	if ok and mouse then
		return NAmanage.SafeCloneRef(mouse)
	end
	return nil
end

NAmanage.uiObj = NAmanage.uiObj or function(v, seen)
	if typeof(v) == "Instance" and v:IsA("ScreenGui") then
		return v
	end

	if type(v) == "function" then
		local ok, res = pcall(v)
		if ok then
			return NAmanage.uiObj(res, seen)
		end
		return nil
	end

	if type(v) == "table" then
		seen = seen or {}
		if seen[v] then
			return nil
		end
		seen[v] = true

		const keys = {
			"ScreenGui",
			"screenGui",
			"Gui",
			"gui",
			"UI",
			"ui",
			"Instance",
			"instance",
		}

		for i = 1, #keys do
			const gui = NAmanage.uiObj(rawget(v, keys[i]), seen)
			if gui then
				return gui
			end
		end
	end

	return nil
end

NAmanage.getUI = NAmanage.getUI or function()
	local gui = NAmanage.uiObj(NAStuff and NAStuff.NASCREENGUI)
	if gui then
		return gui
	end

	if _na_env then
		gui = NAmanage.uiObj(rawget(_na_env, "NA_UI_INSTANCE"))
			or NAmanage.uiObj(rawget(_na_env, "NA_RAW_UI"))
			or NAmanage.uiObj(rawget(_na_env, "NA_UI"))
		if gui then
			return gui
		end
	end

	if _na_shared then
		gui = NAmanage.uiObj(rawget(_na_shared, "NA_UI_INSTANCE"))
			or NAmanage.uiObj(rawget(_na_shared, "NA_RAW_UI"))
			or NAmanage.uiObj(rawget(_na_shared, "NA_UI"))
		if gui then
			return gui
		end
	end

	return nil
end

NAmanage.IsGuiActuallyVisible = NAmanage.IsGuiActuallyVisible or function(inst)
	if typeof(inst) ~= "Instance" then return false end
	local current = inst
	while typeof(current) == "Instance" do
		local okGui, isGui = pcall(function() return current:IsA("GuiObject") end)
		if okGui and isGui and current.Visible == false then return false end
		local okLayer, isLayer = pcall(function() return current:IsA("LayerCollector") end)
		if okLayer and isLayer and current.Enabled == false then return false end
		if current == game then break end
		current = current.Parent
	end
	return true
end

NAmanage.IsUIWindowVisible = NAmanage.IsUIWindowVisible or function(window)
	local frame = window
	if type(window) == "string" then frame = NAUIMANAGER and NAUIMANAGER[window] or nil end
	return NAmanage.IsGuiActuallyVisible(frame)
end

NAmanage.IsLowEndUI = NAmanage.IsLowEndUI or function()
	if NAStuff and NAStuff.LowEndMode == true then return true end
	local uis
	pcall(function() uis = Services.UserInputService or (SafeGetService and SafeGetService("UserInputService")) end)
	if uis and uis.TouchEnabled and not (uis.KeyboardEnabled or uis.MouseEnabled) then return true end
	local ok, size = pcall(function() return Services.Workspace and Services.Workspace.CurrentCamera and Services.Workspace.CurrentCamera.ViewportSize end)
	return ok and size and (size.X <= 900 or size.Y <= 540) or false
end

NAmanage.SetSettingsCanvasDormant = NAmanage.SetSettingsCanvasDormant or function(dormant)
	if not (NAUIMANAGER and NAUIMANAGER.SettingsFrame) then return end
	const autoSize = dormant and Enum.AutomaticSize.None or Enum.AutomaticSize.Y
	const targets = {}
	if NAUIMANAGER.SettingsList then targets[#targets + 1] = NAUIMANAGER.SettingsList end
	if NAgui and NAgui.TabManager and type(NAgui.TabManager.tabs) == "table" then
		for _, info in NAgui.TabManager.tabs do
			if info and info.page then targets[#targets + 1] = info.page end
		end
	end
	for i = 1, #targets do
		const target = targets[i]
		if typeof(target) == "Instance" then
			pcall(function() target.AutomaticCanvasSize = autoSize end)
		end
	end
end

NAmanage.OnUIWindowHidden = NAmanage.OnUIWindowHidden or function(frame)
	if typeof(frame) ~= "Instance" then return end
	if NAUIMANAGER and frame == NAUIMANAGER.SettingsFrame then
		if NAmanage.SettingsScroll then
			if type(NAmanage.SettingsScroll.stopArrowHold) == "function" then pcall(NAmanage.SettingsScroll.stopArrowHold) end
			if type(NAmanage.SettingsScroll.stopThumbDrag) == "function" then pcall(NAmanage.SettingsScroll.stopThumbDrag) end
			if type(NAmanage.SettingsScroll.hide) == "function" then pcall(NAmanage.SettingsScroll.hide) end
		end
	end
end

NAmanage.OnUIWindowShown = NAmanage.OnUIWindowShown or function(frame)
	if typeof(frame) ~= "Instance" then return end
	if NAUIMANAGER and NAgui then
		local lazyMenuKey = nil
		local lazyMenuBinder = nil
		if frame == NAUIMANAGER.chatLogsFrame and type(NAgui.menuv3) == "function" then
			lazyMenuKey = "chatLogsFrame"
			lazyMenuBinder = function() NAgui.menuv3(frame) end
		elseif frame == NAUIMANAGER.NAconsoleFrame and type(NAgui.menuv2) == "function" then
			lazyMenuKey = "NAconsoleFrame"
			lazyMenuBinder = function() NAgui.menuv2(frame) end
		elseif frame == NAUIMANAGER.commandsFrame and type(NAgui.menuv3) == "function" then
			lazyMenuKey = "commandsFrame"
			lazyMenuBinder = function() NAgui.menuv3(frame) end
		elseif frame == NAUIMANAGER.SettingsFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "SettingsFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.CommandKeybindsFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "CommandKeybindsFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.WaypointFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "WaypointFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.BindersFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "BindersFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.ExecutorFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "ExecutorFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.NotepadFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "NotepadFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.MusicFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "MusicFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.ScriptHubFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "ScriptHubFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		elseif frame == NAUIMANAGER.SubplaceViewerFrame and type(NAgui.menu) == "function" then
			lazyMenuKey = "SubplaceViewerFrame"
			lazyMenuBinder = function() NAgui.menu(frame) end
		end
		if lazyMenuKey and lazyMenuBinder and NAStuff["LazyMenuBound_"..lazyMenuKey] ~= frame then
			NAStuff["LazyMenuBound_"..lazyMenuKey] = frame
			const wasVisible = frame.Visible == true
			pcall(lazyMenuBinder)
			if wasVisible and frame.Parent then
				frame.Visible = true
				const body = frame:FindFirstChild("Container")
				if body and body:IsA("GuiObject") and NAmanage.GetAttr(frame, "NAMenuMinimized") ~= true then
					body.Visible = true
				end
			end
		end
	end
	if NAUIMANAGER and frame == NAUIMANAGER.SettingsFrame then
		
