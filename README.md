```lua
script_name("Script")
script_version("2.3")
script_version_number(23)
script_author("Modified")
script_description("Script")
script_properties('work-in-pause')

require "lib.sampfuncs"
require "lib.moonloader"
local vkeys = require 'lib.vkeys'
local ffi = require "ffi"
local vector = require "vector3d"
local sampev = require('samp.events')
local encoding = require('encoding')

encoding.default = 'CP1251'

local getbonePosition = ffi.cast("int (__thiscall*)(void*, float*, int, bool)", 0x5E4280)

local stun_anims = {
    'DAM_armL_frmBK', 'DAM_armL_frmFT', 'DAM_armL_frmLT', 'DAM_armR_frmBK', 'DAM_armR_frmFT', 'DAM_armR_frmRT',
    'DAM_LegL_frmBK', 'DAM_LegL_frmFT', 'DAM_LegL_frmLT', 'DAM_LegR_frmBK', 'DAM_LegR_frmFT', 'DAM_LegR_frmRT',
    'DAM_stomach_frmBK', 'DAM_stomach_frmFT', 'DAM_stomach_frmLT', 'DAM_stomach_frmRT'
}

local skinToGang = {
    [114] = 1, [115] = 1, [116] = 1,
    [105] = 2, [106] = 2, [107] = 2, [269] = 2, [270] = 2, [271] = 2,
    [102] = 3, [103] = 3, [104] = 3, [195] = 3,
    [108] = 4, [109] = 4, [110] = 4,
    [173] = 5, [174] = 5, [175] = 5,
    [117] = 6, [118] = 6, [120] = 6,
    [121] = 7, [122] = 7, [123] = 7,
    [111] = 8, [112] = 8, [113] = 8, [124] = 8, [125] = 8, [126] = 8, [127] = 8
}

local scriptActive = false
local smooth = 22.0
local radius = 6.5
local visibleCheck = true
local checkStuned = true
local checkPause = true
local idFilter = false
local excludeIDsBuffer = "-1"
local heightOffset = 0.0
local useHeightOffset = true
local fovAdjust = 70.0
local disableOnZoom = false
local noFalseAim = true
local modelFilter = false
local zOffset = 0.0
local useZOffset = true

local lastTargetHandle = -1
local killDelayTimer = 0
local shootAimTimer = 0

TAG = {
    TYPE_INFO      = 1,
    TYPE_DEBUG     = 2,
    TYPE_ERROR     = 3,
    TYPE_WARN      = 4,
    TYPE_SYSTEM    = 5,
    TYPE_FATAL     = 6,
    TYPE_EXCEPTION = 7
}

logDebugMessages = false
COLOR_MSG       = 0xC0C0C0
COLOR_SCRIPTMSG = 0x7DD156
COLOR_SENDER    = 0xE0E0E0

function fix(angle)
    if angle > math.pi then angle = angle - (math.pi * 2)
    elseif angle < -math.pi then angle = angle + (math.pi * 2) end
    return angle
end

function stringToIdList(str)
    local ids = {}
    for id in str:gmatch("[^,]+") do
        local num = tonumber(id:match("%d+"))
        if num then table.insert(ids, num) end
    end
    return ids
end

local function isSameSkinGroup(skin1, skin2)
    if skin1 == skin2 then return true end
    local g1 = skinToGang[skin1]
    local g2 = skinToGang[skin2]
    return g1 ~= nil and g1 == g2
end

function getBodyPartCoordinates(id, handle)
    if doesCharExist(handle) then
        local ptr = getCharPointer(handle)
        if ptr ~= 0 then
            local pos = ffi.new("float[3]")
            getbonePosition(ffi.cast("void*", ptr), pos, id, true)
            return true, vector(pos[0], pos[1], pos[2])
        end
    end
    return false
end

function GetNearestBone(handle)
    if useHeightOffset then
        return 3
    else
        local maxDist = 20000    
        local nearestBone = -1
        local bone = {42, 52, 23, 33, 3, 22, 32, 8}
        for n = 1, #bone do
            local crosshairPos = {convertGameScreenCoordsToWindowScreenCoords(339.1, 179.1)}
            local res, bonePosVec = getBodyPartCoordinates(bone[n], handle)
            if res then
                local enPos = {convert3DCoordsToScreen(bonePosVec.x, bonePosVec.y, bonePosVec.z)}
                local distance = math.sqrt((math.pow((enPos[1] - crosshairPos[1]), 2) + math.pow((enPos[2] - crosshairPos[2]), 2)))
                if distance < maxDist then
                    nearestBone = bone[n]
                    maxDist = distance
                end
            end
        end
        return nearestBone == -1 and 3 or nearestBone
    end
end

function GetTargetCoords(id, handle)
    local res, vec = getBodyPartCoordinates(id, handle)
    if res then
        local totalZOffset = 0.0
        if useHeightOffset then totalZOffset = totalZOffset + heightOffset end
        if useZOffset then totalZOffset = totalZOffset + zOffset end
        
        return vec.x, vec.y, vec.z + totalZOffset
    end
    return 0, 0, 0
end

function CheckStuned()
    for k, v in pairs(stun_anims) do
        if isCharPlayingAnim(PLAYER_PED, v) then return false end
    end
    return true
end

function GetNearestPed(fov)
    local max3DDistance = 35.0
    local minCrosshairDist = fov
    local nearestPED = -1
    local excludedIDs = idFilter and stringToIdList(excludeIDsBuffer) or {}
    local mySkin = getCharModel(PLAYER_PED)
    
    for i = 0, sampGetMaxPlayerId(true) do
        if sampIsPlayerConnected(i) then
            local shouldExclude = false
            for _, excludedID in ipairs(excludedIDs) do
                if i == excludedID then shouldExclude = true; break end
            end

            if not (idFilter and shouldExclude) then
                local find, handle = sampGetCharHandleBySampPlayerId(i)
                if find and isCharOnScreen(handle) and not isCharDead(handle) then
                    local targetSkin = getCharModel(handle)
                    if not (modelFilter and isSameSkinGroup(mySkin, targetSkin)) then
                        local enX, enY, enZ = GetTargetCoords(GetNearestBone(handle), handle)
                        local myPosX, myPosY, myPosZ = getCharCoordinates(PLAYER_PED)
                        local distToPlayer = math.sqrt((enX - myPosX)^2 + (enY - myPosY)^2 + (enZ - myPosZ)^2)
                        
                        if distToPlayer <= max3DDistance then
                            local myX, myY, myZ = getActiveCameraCoordinates()
                            local vector = {myX - enX, myY - enY, myZ - enZ}
                            local heightDiff = math.abs(myZ - enZ)
                            local adjustedFov = fov * (1 + heightDiff / 10)
                            local fovFactor = fovAdjust / 70.0
                            local coefficentZ = isWidescreenOnInOptions() and (0.0778 * fovFactor) or (0.103 * fovFactor)
                            
                            local angle = {
                                (math.atan2(vector[2], vector[1]) + 0.04253),
                                (math.atan2((math.sqrt((math.pow(vector[1], 2) + math.pow(vector[2], 2)))), vector[3]) - math.pi / 2 - coefficentZ)
                            }
                            local view = {fix(representIntAsFloat(readMemory(0xB6F258, 4, false))), fix(representIntAsFloat(readMemory(0xB6F248, 4, false)))}
                            
                            local distance = math.sqrt((math.pow(fix(angle[1] - view[1]), 2) + math.pow(fix(angle[2] - view[2]), 2))) * 57.2957795131
                            
                            if distance <= adjustedFov and distance < minCrosshairDist then
                                minCrosshairDist = distance
                                nearestPED = handle
                            end
                        end
                    end
                end
            end
        end
    end
    return nearestPED
end

function No_false_aim(targetHandle)
    if not noFalseAim then return true end

    local now = os.clock()

    if targetHandle == -1 then
        lastTargetHandle = -1
        return true
    end

    if lastTargetHandle ~= -1 and (not doesCharExist(lastTargetHandle) or isCharDead(lastTargetHandle)) then
        killDelayTimer = now + 0.3
        lastTargetHandle = -1
        return false
    end

    if now < killDelayTimer then
        return false
    end

    if lastTargetHandle ~= -1 and targetHandle ~= lastTargetHandle then
        killDelayTimer = now + 0.25
        lastTargetHandle = targetHandle
        return false
    end

    lastTargetHandle = targetHandle
    return true
end

function sampev.onSendBulletSync(data)
    if scriptActive then
        shootAimTimer = os.clock() + 0.25
    end
end

function process_logic()
    if scriptActive and os.clock() < shootAimTimer then
        local isAiming = getCurrentCharWeapon(PLAYER_PED) ~= 0
        local weaponId = getCurrentCharWeapon(PLAYER_PED)
        
        if disableOnZoom and isAiming and (weaponId == 30 or weaponId == 31) then
            return
        end

        local handle = GetNearestPed(radius)

        if not No_false_aim(handle) then
            return
        end

        if handle ~= -1 then
            local _, playerID = sampGetPlayerIdByCharHandle(handle)
            local result = sampIsPlayerConnected(playerID)
            if result then
                if (checkStuned and not CheckStuned()) then return end
                if (checkPause and sampIsPlayerPaused(playerID)) then return end

                local myX, myY, myZ = getActiveCameraCoordinates()
                local enX, enY, enZ = GetTargetCoords(GetNearestBone(handle), handle)

                if not visibleCheck or (visibleCheck and isLineOfSightClear(myX, myY, myZ, enX, enY, enZ, true, true, false, true, true)) then
                    local vector = {myX - enX, myY - enY, myZ - enZ}
                    local fovFactor = fovAdjust / 70.0
                    local coefficentZ = isWidescreenOnInOptions() and (0.0778 * fovFactor) or (0.103 * fovFactor)

                    local angle = {
                        (math.atan2(vector[2], vector[1]) + 0.04253),
                        (math.atan2((math.sqrt((math.pow(vector[1], 2) + math.pow(vector[2], 2)))), vector[3]) - math.pi / 2 - coefficentZ)
                    }

                    local view = {
                        fix(representIntAsFloat(readMemory(0xB6F258, 4, false))),
                        fix(representIntAsFloat(readMemory(0xB6F248, 4, false)))
                    }

                    local difference = {
                        fix(angle[1] - view[1]),
                        fix(angle[2] - view[2])
                    }

                    local stepX = difference[1] / smooth
                    local stepY = difference[2] / smooth

                    local maxDelta = 0.022 
                    if math.abs(stepX) > maxDelta then stepX = (stepX > 0) and maxDelta or -maxDelta end
                    if math.abs(stepY) > maxDelta then stepY = (stepY > 0) and maxDelta or -maxDelta end

                    setCameraPositionUnfixed(view[2] + stepY, view[1] + stepX)
                end
            end
        end
    else
        lastTargetHandle = -1
        killDelayTimer = 0
    end
end

function isMonser01()
    return true
end

function main()
    if not isSampfuncsLoaded() then return end
    while not isSampAvailable() do wait(0) end

    sampfuncsRegisterConsoleCommand("lua", do_lua)
    sampfuncsRegisterConsoleCommand(">>", do_lua)

    sampRegisterChatCommand("tws", function()
        scriptActive = not scriptActive
    end)

    lua_thread.create(function()
        while true do
            wait(0)
            process_logic()
        end
    end)

    wait(-1)
end

function log_message(msg, tagtext, tagcolor, sender)
    local str = string.format("{%06X}[ML] ", COLOR_MSG)
    if tagtext then str = str .. string.format("{%06X}(%s) ", tagcolor, tagtext) end
    if sender then str = str .. string.format("{%06X}%s: ", COLOR_SENDER, sender.name) end
    sampfuncsLog(string.format("%s{%06X}%s", str, COLOR_MSG, msg))
end

function do_lua(code)
    if code:sub(1,1) == '=' then code = "print(" .. code:sub(2, -1) .. ")" end
    local func, err = load(code)
    if func then
        local result, err = pcall(func)
        if not result then onSystemMessage(err, TAG.TYPE_ERROR, thisScript()) end
    else
        onSystemMessage(err, TAG.TYPE_ERROR, thisScript())
    end
end

function onSystemMessage(msg, type, sender)
    if isSampfuncsLoaded() and isOpcodesAvailable() and (type ~= TAG.TYPE_DEBUG or logDebugMessages) then
        local tagtxt = get_tag_text(type)
        local tagclr = get_tag_color(type) or COLOR_MSG
        log_message(msg, tagtxt, tagclr, sender)
    end
end

local tags = {
    [TAG.TYPE_INFO] =      {"info", 0xA9EFF5},
    [TAG.TYPE_DEBUG] =     {"debug", 0xAFA9F5},
    [TAG.TYPE_ERROR] =     {"error", 0xFF7070},
    [TAG.TYPE_WARN] =      {"warn", 0xF5C28E},
    [TAG.TYPE_SYSTEM] =    {"system", 0xFA9746},
    [TAG.TYPE_FATAL] =     {"fatal", 0x040404},
    [TAG.TYPE_EXCEPTION] = {"exception", 0xF5A9A9}
}

function get_tag_text(n) return tags[n] and tags[n][1] or nil end
function get_tag_color(n) return tags[n] and tags[n][2] or nil end

```
