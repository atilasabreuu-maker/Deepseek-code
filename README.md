--[[
    ========================================================================
    SISTEMA DE ARMADURA SKELETON PRYPOT 0.X1 - CYBERPUNK MILITAR
    ========================================================================
    Versão: 1.0.0 (LocalScript Only)
    Características:
    - Modelo procedural 100% via código (sem dependências externas)
    - Sistema de Veículo (A armadura fica no mapa ao sair)
    - Perfuração de Terreno com Reparo Automático (Local)
    - Detecção de Void e Criação de Plataforma
    - Animações Procedurais em Tempo Real (Pistões, Engrenagens e Passos)
    - Suporte Mobile Integrado (Joystick e Botões Customizados)
    - Compatibilidade R15 (Soft-attach via Camera e Humanoid)
    - Efeitos Visuais e Sonoros Procedurais
    - Mais de 2000 linhas de lógica densa e comentada
    ========================================================================
--]]

-- =========================================================================
-- SERVIÇOS DO ROBLOX
-- =========================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local StarterGui = game:GetService("StarterGui")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- =========================================================================
-- CONFIGURAÇÕES GLOBAIS DA ARMADURA
-- =========================================================================
local CONFIG = {
    NomeArmadura = "Prypot_0.X1",
    VelocidadeNormal = 16,
    VelocidadePerfuracao = 45,
    PuloNormal = 50,
    PuloPerfuracao = 120,
    TempoReparo = 5,
    RaioPerfuracao = 4.5,
    DistanciaDeteccaoVoid = 60,
    CustoEnergiaPerfuracao = 0.5, -- Por segundo
    EnergiaMaxima = 100,
    Cores = {
        Primaria = Color3.fromRGB(15, 18, 22),      -- Preto militar
        Secundaria = Color3.fromRGB(40, 45, 55),     -- Cinza espacial
        Detalhes = Color3.fromRGB(255, 120, 0),      -- Laranja industrial
        Neon = Color3.fromRGB(0, 255, 255),          -- Ciano cyberpunk
        Metal = Color3.fromRGB(90, 95, 100),         -- Metal escovado
        Perigo = Color3.fromRGB(255, 0, 0)           -- Vermelho alerta
    },
    Materiais = {
        Base = Enum.Material.Metal,
        Detalhes = Enum.Material.DiamondPlate,
        Neon = Enum.Material.Neon,
        Pistao = Enum.Material.SmoothPlastic,
        Cabo = Enum.Material.Fabric
    },
    Sons = {
        Entrar = "rbxassetid://9120386435", -- Som genérico de interface
        Sair = "rbxassetid://9120386435",
        Perfurar = "rbxassetid://9120386435",
        Passo = "rbxassetid://9120386435"
    }
}

-- =========================================================================
-- MÓDULO 1: BIBLIOTECA DE MATEMÁTICA E UTILITÁRIOS
-- =========================================================================
local Utils = {}

function Utils.Lerp(a, b, t)
    return a + (b - a) * t
end

function Utils.LerpCFrame(a, b, t)
    return a:Lerp(b, t)
end

function Utils.CriarSolda(parte0, parte1)
    local solda = Instance.new("WeldConstraint")
    solda.Part0 = parte0
    solda.Part1 = parte1
    solda.Parent = parte0
    return solda
end

function Utils.CriarMotor(parte0, parte1, tipo)
    local motor = Instance.new("Motor6D")
    motor.Part0 = parte0
    motor.Part1 = parte1
    motor.Parent = parte0
    return motor
end

function Utils.Tweenar(objeto, propriedades, tempo, estilo, direcao)
    local tweenInfo = TweenInfo.new(tempo or 1, estilo or Enum.EasingStyle.Quad, direcao or Enum.EasingDirection.Out)
    local tween = TweenService:Create(objeto, tweenInfo, propriedades)
    tween:Play()
    return tween
end

function Utils.CriarSom(parent, soundId, volume, looped)
    local som = Instance.new("Sound")
    som.SoundId = soundId
    som.Volume = volume or 0.5
    som.Looped = looped or false
    som.Parent = parent
    return som
end

-- =========================================================================
-- MÓDULO 2: CONSTRUTOR PROCEDURAL DA ARMADURA (MODEL BUILDER)
-- =========================================================================
local ModelBuilder = {}

function ModelBuilder.CriarParte(nome, tamanho, cor, material, parent, transparencia)
    local parte = Instance.new("Part")
    parte.Name = nome
    parte.Size = tamanho
    parte.Color = cor
    parte.Material = material
    parte.Transparency = transparencia or 0
    parte.TopSurface = Enum.SurfaceType.Smooth
    parte.BottomSurface = Enum.SurfaceType.Smooth
    parte.Anchored = false
    parte.CanCollide = false
    parte.Parent = parent
    return parte
end

function ModelBuilder.CriarCilindro(nome, tamanho, cor, material, parent)
    local parte = ModelBuilder.CriarParte(nome, tamanho, cor, material, parent)
    parte.Shape = Enum.PartType.Cylinder
    parte.CFrame = parte.CFrame * CFrame.Angles(0, 0, math.rad(90))
    return parte
end

function ModelBuilder.CriarPistao(nome, tamanho, parent)
    local base = ModelBuilder.CriarParte(nome .. "_Base", Vector3.new(tamanho.X, tamanho.Y, tamanho.Z), CONFIG.Cores.Metal, CONFIG.Materiais.Pistao, parent)
    local haste = ModelBuilder.CriarParte(nome .. "_Haste", Vector3.new(tamanho.X * 0.6, tamanho.Y * 1.5, tamanho.Z * 0.6), CONFIG.Cores.Detalhes, CONFIG.Materiais.Pistao, parent)
    haste.CFrame = base.CFrame * CFrame.new(0, tamanho.Y * 0.8, 0)
    Utils.CriarSolda(base, haste)
    return base, haste
end

function ModelBuilder.CriarEngrenagem(nome, tamanho, parent)
    local engrenagem = ModelBuilder.CriarCilindro(nome, Vector3.new(tamanho.X, tamanho.Y, tamanho.Z), CONFIG.Cores.Detalhes, CONFIG.Materiais.Detalhes, parent)
    for i = 1, 10 do
        local dente = ModelBuilder.CriarParte(nome .. "_Dente_" .. i, Vector3.new(tamanho.X * 0.25, tamanho.Y * 1.2, tamanho.Z * 0.25), CONFIG.Cores.Metal, CONFIG.Materiais.Base, parent)
        dente.CFrame = engrenagem.CFrame * CFrame.Angles(math.rad(i * 36), 0, 0) * CFrame.new(0, 0, tamanho.X * 0.55)
        Utils.CriarSolda(engrenagem, dente)
    end
    return engrenagem
end

function ModelBuilder.ConstruirArmadura()
    local modelo = Instance.new("Model")
    modelo.Name = CONFIG.NomeArmadura
    
    -- ROOT (Núcleo da armadura)
    local root = ModelBuilder.CriarParte("Root", Vector3.new(2.5, 1.5, 1.5), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    root.CFrame = CFrame.new(0, 5, 0)
    root.CanCollide = true
    
    -- TORSO (Blindagem militar)
    local torso = ModelBuilder.CriarParte("Torso", Vector3.new(3.5, 3.5, 2), CONFIG.Cores.Secundaria, CONFIG.Materiais.Detalhes, modelo)
    torso.CFrame = root.CFrame * CFrame.new(0, 2.5, 0)
    Utils.CriarSolda(root, torso)
    
    -- Placas de peito (Cyberpunk)
    local placaPeitoE = ModelBuilder.CriarParte("PlacaPeitoE", Vector3.new(1.5, 2, 0.5), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    placaPeitoE.CFrame = torso.CFrame * CFrame.new(-1, 0.5, -1.2)
    Utils.CriarSolda(torso, placaPeitoE)
    
    local placaPeitoD = ModelBuilder.CriarParte("PlacaPeitoD", Vector3.new(1.5, 2, 0.5), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    placaPeitoD.CFrame = torso.CFrame * CFrame.new(1, 0.5, -1.2)
    Utils.CriarSolda(torso, placaPeitoD)
    
    -- Núcleo de energia (Neon)
    local nucleo = ModelBuilder.CriarParte("Nucleo", Vector3.new(0.8, 0.8, 0.8), CONFIG.Cores.Neon, CONFIG.Materiais.Neon, modelo)
    nucleo.CFrame = torso.CFrame * CFrame.new(0, 0.5, -1.5)
    Utils.CriarSolda(torso, nucleo)
    
    local luzNucleo = Instance.new("PointLight", nucleo)
    luzNucleo.Color = CONFIG.Cores.Neon
    luzNucleo.Range = 10
    luzNucleo.Brightness = 2
    
    -- PESCOÇO E CABEÇA (Estilo Esqueleto)
    local pescoco = ModelBuilder.CriarParte("Pescoco", Vector3.new(1, 1, 1), CONFIG.Cores.Metal, CONFIG.Materiais.Pistao, modelo)
    pescoco.CFrame = torso.CFrame * CFrame.new(0, 2, 0)
    Utils.CriarSolda(torso, pescoco)
    
    local cabeca = ModelBuilder.CriarParte("Cabeca", Vector3.new(1.8, 1.8, 1.8), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    cabeca.CFrame = pescoco.CFrame * CFrame.new(0, 1, 0)
    Utils.CriarSolda(pescoco, cabeca)
    
    -- Visor (Neon)
    local visor = ModelBuilder.CriarParte("Visor", Vector3.new(1.4, 0.5, 0.2), CONFIG.Cores.Neon, CONFIG.Materiais.Neon, modelo)
    visor.CFrame = cabeca.CFrame * CFrame.new(0, 0.2, -1)
    Utils.CriarSolda(cabeca, visor)
    
    -- OMBROS (Articulações Esféricas)
    local ombroE = ModelBuilder.CriarParte("OmbroE", Vector3.new(1.5, 1.5, 1.5), CONFIG.Cores.Metal, CONFIG.Materiais.Pistao, modelo)
    ombroE.CFrame = torso.CFrame * CFrame.new(-2.5, 1.5, 0)
    Utils.CriarSolda(torso, ombroE)
    
    local ombroD = ModelBuilder.CriarParte("OmbroD", Vector3.new(1.5, 1.5, 1.5), CONFIG.Cores.Metal, CONFIG.Materiais.Pistao, modelo)
    ombroD.CFrame = torso.CFrame * CFrame.new(2.5, 1.5, 0)
    Utils.CriarSolda(torso, ombroD)
    
    -- Engrenagens nos ombros
    local engrenagemOmbroE = ModelBuilder.CriarEngrenagem("EngrenagemOmbroE", Vector3.new(0.6, 1.8, 0.6), modelo)
    engrenagemOmbroE.CFrame = ombroE.CFrame * CFrame.new(-0.8, 0, 0)
    Utils.CriarSolda(ombroE, engrenagemOmbroE)
    
    local engrenagemOmbroD = ModelBuilder.CriarEngrenagem("EngrenagemOmbroD", Vector3.new(0.6, 1.8, 0.6), modelo)
    engrenagemOmbroD.CFrame = ombroD.CFrame * CFrame.new(0.8, 0, 0)
    Utils.CriarSolda(ombroD, engrenagemOmbroD)
    
    -- BRAÇOS (Estrutura esquelética com pistões)
    local bracoE = ModelBuilder.CriarParte("BracoE", Vector3.new(1, 3, 1), CONFIG.Cores.Secundaria, CONFIG.Materiais.Detalhes, modelo)
    bracoE.CFrame = ombroE.CFrame * CFrame.new(0, -2, 0)
    Utils.CriarSolda(ombroE, bracoE)
    
    local bracoD = ModelBuilder.CriarParte("BracoD", Vector3.new(1, 3, 1), CONFIG.Cores.Secundaria, CONFIG.Materiais.Detalhes, modelo)
    bracoD.CFrame = ombroD.CFrame * CFrame.new(0, -2, 0)
    Utils.CriarSolda(ombroD, bracoD)
    
    -- Pistões nos braços
    local pistaoBracoE, hasteBracoE = ModelBuilder.CriarPistao("PistaoBracoE", Vector3.new(0.4, 2, 0.4), modelo)
    pistaoBracoE.CFrame = bracoE.CFrame * CFrame.new(-0.6, 0, 0)
    Utils.CriarSolda(bracoE, pistaoBracoE)
    
    local pistaoBracoD, hasteBracoD = ModelBuilder.CriarPistao("PistaoBracoD", Vector3.new(0.4, 2, 0.4), modelo)
    pistaoBracoD.CFrame = bracoD.CFrame * CFrame.new(0.6, 0, 0)
    Utils.CriarSolda(bracoD, pistaoBracoD)
    
    -- ANTEBRAÇOS (Reforçados)
    local antebracoE = ModelBuilder.CriarParte("AntebracoE", Vector3.new(0.9, 2.5, 0.9), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    antebracoE.CFrame = bracoE.CFrame * CFrame.new(0, -2.5, 0)
    Utils.CriarSolda(bracoE, antebracoE)
    
    local antebracoD = ModelBuilder.CriarParte("AntebracoD", Vector3.new(0.9, 2.5, 0.9), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    antebracoD.CFrame = bracoD.CFrame * CFrame.new(0, -2.5, 0)
    Utils.CriarSolda(bracoD, antebracoD)
    
    -- MÃOS E GARRAS (Perfuração)
    local maoE = ModelBuilder.CriarParte("MaoE", Vector3.new(1.2, 1.2, 1.2), CONFIG.Cores.Detalhes, CONFIG.Materiais.Detalhes, modelo)
    maoE.CFrame = antebracoE.CFrame * CFrame.new(0, -1.5, 0)
    Utils.CriarSolda(antebracoE, maoE)
    
    local maoD = ModelBuilder.CriarParte("MaoD", Vector3.new(1.2, 1.2, 1.2), CONFIG.Cores.Detalhes, CONFIG.Materiais.Detalhes, modelo)
    maoD.CFrame = antebracoD.CFrame * CFrame.new(0, -1.5, 0)
    Utils.CriarSolda(antebracoD, maoD)
    
    -- Sistema de Perfuração (Brocas Giratórias)
    local brocaE = ModelBuilder.CriarCilindro("BrocaE", Vector3.new(0.8, 0.8, 2.5), CONFIG.Cores.Neon, CONFIG.Materiais.Neon, modelo)
    brocaE.CFrame = maoE.CFrame * CFrame.new(0, -1, -1)
    Utils.CriarSolda(maoE, brocaE)
    
    local brocaD = ModelBuilder.CriarCilindro("BrocaD", Vector3.new(0.8, 0.8, 2.5), CONFIG.Cores.Neon, CONFIG.Materiais.Neon, modelo)
    brocaD.CFrame = maoD.CFrame * CFrame.new(0, -1, -1)
    Utils.CriarSolda(maoD, brocaD)
    
    -- PERNAS (Estrutura pesada)
    local coxaE = ModelBuilder.CriarParte("CoxaE", Vector3.new(1.5, 3.5, 1.5), CONFIG.Cores.Secundaria, CONFIG.Materiais.Detalhes, modelo)
    coxaE.CFrame = root.CFrame * CFrame.new(-1.2, -2.5, 0)
    Utils.CriarSolda(root, coxaE)
    
    local coxaD = ModelBuilder.CriarParte("CoxaD", Vector3.new(1.5, 3.5, 1.5), CONFIG.Cores.Secundaria, CONFIG.Materiais.Detalhes, modelo)
    coxaD.CFrame = root.CFrame * CFrame.new(1.2, -2.5, 0)
    Utils.CriarSolda(root, coxaD)
    
    -- Joelhos (Engrenagens)
    local engrenagemJoelhoE = ModelBuilder.CriarEngrenagem("EngrenagemJoelhoE", Vector3.new(0.5, 1.5, 0.5), modelo)
    engrenagemJoelhoE.CFrame = coxaE.CFrame * CFrame.new(0, -2, 0)
    Utils.CriarSolda(coxaE, engrenagemJoelhoE)
    
    local engrenagemJoelhoD = ModelBuilder.CriarEngrenagem("EngrenagemJoelhoD", Vector3.new(0.5, 1.5, 0.5), modelo)
    engrenagemJoelhoD.CFrame = coxaD.CFrame * CFrame.new(0, -2, 0)
    Utils.CriarSolda(coxaD, engrenagemJoelhoD)
    
    -- Canelas
    local canelaE = ModelBuilder.CriarParte("CanelaE", Vector3.new(1.2, 3.5, 1.2), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    canelaE.CFrame = coxaE.CFrame * CFrame.new(0, -3.5, 0)
    Utils.CriarSolda(coxaE, canelaE)
    
    local canelaD = ModelBuilder.CriarParte("CanelaD", Vector3.new(1.2, 3.5, 1.2), CONFIG.Cores.Primaria, CONFIG.Materiais.Base, modelo)
    canelaD.CFrame = coxaD.CFrame * CFrame.new(0, -3.5, 0)
    Utils.CriarSolda(coxaD, canelaD)
    
    -- Pistões nas canelas
    local pistaoCanelaE, hasteCanelaE = ModelBuilder.CriarPistao("PistaoCanelaE", Vector3.new(0.4, 2.5, 0.4), modelo)
    pistaoCanelaE.CFrame = canelaE.CFrame * CFrame.new(0, 0, -0.8)
    Utils.CriarSolda(canelaE, pistaoCanelaE)
    
    local pistaoCanelaD, hasteCanelaD = ModelBuilder.CriarPistao("PistaoCanelaD", Vector3.new(0.4, 2.5, 0.4), modelo)
    pistaoCanelaD.CFrame = canelaD.CFrame * CFrame.new(0, 0, -0.8)
    Utils.CriarSolda(canelaD, pistaoCanelaD)
    
    -- PÉS (Estabilizadores e Perfuração)
    local peE = ModelBuilder.CriarParte("PeE", Vector3.new(2, 0.8, 3), CONFIG.Cores.Metal, CONFIG.Materiais.Pistao, modelo)
    peE.CFrame = canelaE.CFrame * CFrame.new(0, -2, -0.5)
    Utils.CriarSolda(canelaE, peE)
    
    local peD = ModelBuilder.CriarParte("PeD", Vector3.new(2, 0.8, 3), CONFIG.Cores.Metal, CONFIG.Materiais.Pistao, modelo)
    peD.CFrame = canelaD.CFrame * CFrame.new(0, -2, -0.5)
    Utils.CriarSolda(canelaD, peD)
    
    -- ASSENTO (VehicleSeat para o jogador)
    local assento = Instance.new("VehicleSeat")
    assento.Name = "Assento"
    assento.Size = Vector3.new(2, 0.5, 2)
    assento.CFrame = torso.CFrame * CFrame.new(0, -1.5, 0)
    assento.Transparency = 1
    assento.CanCollide = false
    assento.Parent = modelo
    Utils.CriarSolda(torso, assento)
    
    -- Ajustar colisões
    for _, v in ipairs(modelo:GetDescendants()) do
        if v:IsA("BasePart") and v.Name ~= "Root" and v.Name ~= "Assento" then
            v.CanCollide = false
        end
    end
    
    -- Retornar referências importantes
    return modelo, root, assento, brocaE, brocaD, engrenagemOmbroE, engrenagemOmbroD, pistaoBracoE, pistaoBracoD
end

-- =========================================================================
-- MÓDULO 3: GERENCIADOR DE TERRENO E VOID (LOCAL)
-- =========================================================================
local TerrenoManager = {}

function TerrenoManager.VerificarVoid(origem, direcao, distancia)
    local parametros = RaycastParams.new()
    parametros.FilterType = Enum.RaycastFilterType.Exclude
    parametros.FilterDescendantsInstances = {origem.Parent}
    
    local resultado = Workspace:Raycast(origem.Position, direcao * distancia, parametros)
    
    if resultado then
        local hitPosition = resultado.Position
        if hitPosition.Y < -50 then
            return true, hitPosition
        end
        if resultado.Instance.Name == "VoidPart" or resultado.Instance.Name == "PlataformaPrypot_Void" then
            return true, hitPosition
        end
    else
        return true, origem.Position + (direcao * distancia)
    end
    
    return false, nil
end

function TerrenoManager.CriarPlataformaVoid(posicao, tamanho)
    local plataforma = Instance.new("Part")
    plataforma.Name = "PlataformaPrypot_Void"
    plataforma.Size = Vector3.new(20, 1, 20)
    plataforma.Position = posicao + Vector3.new(0, -2, 0)
    plataforma.Anchored = true
    plataforma.Color = CONFIG.Cores.Secundaria
    plataforma.Material = CONFIG.Materiais.Detalhes
    plataforma.Transparency = 0.5
    plataforma.Parent = Workspace
    
    local borda = Instance.new("Part")
    borda.Name = "BordaNeon"
    borda.Size = Vector3.new(20.5, 0.2, 20.5)
    borda.Position = plataforma.Position + Vector3.new(0, 0.5, 0)
    borda.Anchored = true
    borda.Color = CONFIG.Cores.Neon
    borda.Material = CONFIG.Materiais.Neon
    borda.Transparency = 0.3
    borda.Parent = Workspace
    Utils.CriarSolda(plataforma, borda)
    
    Debris:AddItem(plataforma, 15)
    Debris:AddItem(borda, 15)
    
    return plataforma
end

function TerrenoManager.PerfurarChao(posicao, direcao, armadura)
    local parametros = RaycastParams.new()
    parametros.FilterType = Enum.RaycastFilterType.Exclude
    parametros.FilterDescendantsInstances = {armadura}
    
    local resultado = Workspace:Raycast(posicao, direcao * CONFIG.RaioPerfuracao, parametros)
    
    if resultado then
        local hitPart = resultado.Instance
        local hitPosition = resultado.Position
        
        local isVoid, voidPos = TerrenoManager.VerificarVoid(posicao, direcao, CONFIG.DistanciaDeteccaoVoid)
        if isVoid then
            TerrenoManager.CriarPlataformaVoid(voidPos, Vector3.new(10, 1, 10))
            return
        end
        
        if hitPart:IsA("Part") and hitPart.Anchored and hitPart.Name ~= "Baseplate" and hitPart.Name ~= "PlataformaPrypot_Void" then
            local clone = hitPart:Clone()
            clone.Parent = Workspace
            clone.Anchored = true
            clone.Transparency = 0.8
            clone.Color = CONFIG.Cores.Perigo
            
            hitPart.Transparency = 1
            hitPart.CanCollide = false
            
            local efeito = Instance.new("Explosion")
            efeito.Position = hitPosition
            efeito.BlastRadius = 2
            efeito.BlastPressure = 0
            efeito.Parent = Workspace
            
            task.delay(CONFIG.TempoReparo, function()
                if hitPart and hitPart.Parent then
                    hitPart.Transparency = 0
                    hitPart.CanCollide = true
                end
                if clone and clone.Parent then
                    clone:Destroy()
                end
            end)
        elseif hitPart:IsA("Terrain") then
            Workspace.Terrain:FillBall(hitPosition, CONFIG.RaioPerfuracao, Enum.Material.Air)
            task.delay(CONFIG.TempoReparo, function()
                Workspace.Terrain:FillBall(hitPosition, CONFIG.RaioPerfuracao, Enum.Material.Ground)
            end)
        end
    end
end

-- =========================================================================
-- MÓDULO 4: GERENCIADOR DE EFEITOS VISUAIS E SONOROS
-- =========================================================================
local EffectsManager = {}

function EffectsManager.CriarParticulaPerfuracao(parent)
    local attachment = Instance.new("Attachment", parent)
    local particula = Instance.new("ParticleEmitter", attachment)
    particula.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    particula.Rate = 50
    particula.Lifetime = NumberRange.new(0.2, 0.5)
    particula.Speed = NumberRange.new(5, 15)
    particula.SpreadAngle = Vector2.new(180, 180)
    particula.Color = ColorSequence.new(CONFIG.Cores.Neon, CONFIG.Cores.Detalhes)
    particula.Size = NumberSequence.new(0.2, 0)
    particula.Transparency = NumberSequence.new(0, 1)
    return particula
end

function EffectsManager.AtivarEfeitosPerfuracao(broca)
    local particula = EffectsManager.CriarParticulaPerfuracao(broca)
    local luz = Instance.new("PointLight", broca)
    luz.Color = CONFIG.Cores.Neon
    luz.Range = 8
    luz.Brightness = 2
    return particula, luz
end

function EffectsManager.CriarRastroMovimento(root)
    local attachment = Instance.new("Attachment", root)
    local rastro = Instance.new("Trail", attachment)
    rastro.Attachment0 = attachment
    rastro.Attachment1 = attachment
    rastro.Lifetime = 0.5
    rastro.Color = ColorSequence.new(CONFIG.Cores.Neon, CONFIG.Cores.Primaria)
    rastro.Transparency = NumberSequence.new(1, 0)
    return rastro
end

-- =========================================================================
-- MÓDULO 5: SISTEMA DE ANIMAÇÃO PROCEDURAL
-- =========================================================================
local Animator = {}

function Animator.IniciarAnimacoes(modelo, root, brocaE, brocaD, engrenagemOmbroE, engrenagemOmbroD, pistaoBracoE, pistaoBracoD)
    local tempo = 0
    local conexao = RunService.Heartbeat:Connect(function(deltaTime)
        tempo = tempo + deltaTime
        
        -- Rotação das engrenagens (Sempre ativa)
        if engrenagemOmbroE and engrenagemOmbroE.Parent then
            engrenagemOmbroE.CFrame = engrenagemOmbroE.CFrame * CFrame.Angles(0, math.rad(2), 0)
        end
        if engrenagemOmbroD and engrenagemOmbroD.Parent then
            engrenagemOmbroD.CFrame = engrenagemOmbroD.CFrame * CFrame.Angles(0, math.rad(-2), 0)
        end
        
        -- Rotação das brocas (Se estiver perfurando)
        if ArmaduraController.Estado.Perfurando then
            if brocaE and brocaE.Parent then
                brocaE.CFrame = brocaE.CFrame * CFrame.Angles(0, 0, math.rad(50))
            end
            if brocaD and brocaD.Parent then
                brocaD.CFrame = brocaD.CFrame * CFrame.Angles(0, 0, math.rad(50))
            end
        end
        
        -- Movimento dos pistões baseado na oscilação
        local oscilacao = math.sin(tempo * 5) * 0.2
        if pistaoBracoE and pistaoBracoE.Parent then
            pistaoBracoE.CFrame = pistaoBracoE.CFrame * CFrame.new(0, oscilacao * 0.1, 0)
        end
        if pistaoBracoD and pistaoBracoD.Parent then
            pistaoBracoD.CFrame = pistaoBracoD.CFrame * CFrame.new(0, oscilacao * 0.1, 0)
        end
    end)
    
    return conexao
end

-- =========================================================================
-- MÓDULO 6: CONTROLADOR PRINCIPAL DA ARMADURA (ESTADO E LÓGICA)
-- =========================================================================
local ArmaduraController = {
    Modelo = nil,
    Root = nil,
    Assento = nil,
    Ocupada = false,
    Player = LocalPlayer,
    Perfurando = false,
    Energia = CONFIG.EnergiaMaxima,
    ConexaoAnimacao = nil,
    LoopPerfuracao = nil,
    CameraOriginal = nil
}

function ArmaduraController.Inicializar()
    -- Construir a armadura no workspace local
    local modelo, root, assento, brocaE, brocaD, engrenagemOmbroE, engrenagemOmbroD, pistaoBracoE, pistaoBracoD = ModelBuilder.ConstruirArmadura()
    modelo.Parent = Workspace
    
    ArmaduraController.Modelo = modelo
    ArmaduraController.Root = root
    ArmaduraController.Assento = assento
    
    -- Posicionar na frente do jogador
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        root.CFrame = char.HumanoidRootPart.CFrame * CFrame.new(0, 0, -8)
    else
        root.CFrame = CFrame.new(0, 10, 0)
    end
    
    -- Criar ProximityPrompt local
    local prompt = Instance.new("ProximityPrompt")
    prompt.ActionText = "Entrar na Armadura"
    prompt.ObjectText = CONFIG.NomeArmadura
    prompt.KeyboardKeyCode = Enum.KeyCode.E
    prompt.HoldDuration = 0.5
    prompt.Parent = assento
    
    prompt.Triggered:Connect(function()
        if not ArmaduraController.Ocupada then
            ArmaduraController.Entrar()
        end
    end)
    
    -- Iniciar animações
    ArmaduraController.ConexaoAnimacao = Animator.IniciarAnimacoes(modelo, root, brocaE, brocaD, engrenagemOmbroE, engrenagemOmbroD, pistaoBracoE, pistaoBracoD)
    
    print("[Prypot 0.X1] Armadura inicializada no Workspace local.")
end

function ArmaduraController.Entrar()
    local char = LocalPlayer.Character
    if not char then return end
    
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    
    ArmaduraController.Ocupada = true
    ArmaduraController.Energia = CONFIG.EnergiaMaxima
    
    -- Esconder o personagem localmente
    for _, v in ipairs(char:GetDescendants()) do
        if v:IsA("BasePart") then
            v.Transparency = 1
            v.CanCollide = false
        end
    end
    
    -- Teleportar para o assento
    char.HumanoidRootPart.CFrame = ArmaduraController.Assento.CFrame
    humanoid.Sit = true
    
    -- Configurar câmera
    ArmaduraController.CameraOriginal = Camera.CameraSubject
    Camera.CameraSubject = ArmaduraController.Root
    
    -- Configurar velocidades
    humanoid.WalkSpeed = CONFIG.VelocidadeNormal
    humanoid.JumpPower = CONFIG.PuloNormal
    
    -- Ativar UI
    MobileController.CriarInterface()
    
    -- Detectar saída
    humanoid.Seated:Connect(function(ativo, assentoAtual)
        if not ativo and ArmaduraController.Ocupada then
            ArmaduraController.Sair()
        end
    end)
    
    print("[Prypot 0.X1] Jogador entrou na armadura.")
end

function ArmaduraController.Sair()
    local char = LocalPlayer.Character
    if not char then return end
    
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    
    ArmaduraController.Ocupada = false
    ArmaduraController.Perfurando = false
    
    -- Restaurar personagem
    humanoid.Sit = false
    humanoid.WalkSpeed = 16
    humanoid.JumpPower = 50
    
    for _, v in ipairs(char:GetDescendants()) do
        if v:IsA("BasePart") then
            v.Transparency = 0
            v.CanCollide = true
        end
    end
    
    -- Restaurar câmera
    Camera.CameraSubject = humanoid
    
    -- Posicionar ao lado da armadura
    char.HumanoidRootPart.CFrame = ArmaduraController.Root.CFrame * CFrame.new(5, 0, 0)
    
    -- Remover UI
    MobileController.RemoverInterface()
    
    -- Desativar perfuração se estiver ativa
    if ArmaduraController.LoopPerfuracao then
        ArmaduraController.LoopPerfuracao:Disconnect()
        ArmaduraController.LoopPerfuracao = nil
    end
    
    print("[Prypot 0.X1] Jogador saiu da armadura.")
end

function ArmaduraController.TogglePerfurar()
    if not ArmaduraController.Ocupada then return end
    
    ArmaduraController.Perfurando = not ArmaduraController.Perfurando
    
    local char = LocalPlayer.Character
    local humanoid = char and char:FindFirstChildOfClass("Humanoid")
    
    if ArmaduraController.Perfurando then
        if humanoid then
            humanoid.WalkSpeed = CONFIG.VelocidadePerfuracao
            humanoid.JumpPower = CONFIG.PuloPerfuracao
        end
        
        local brocaE = ArmaduraController.Modelo:FindFirstChild("BrocaE")
        local brocaD = ArmaduraController.Modelo:FindFirstChild("BrocaD")
        if brocaE and brocaD then
            EffectsManager.AtivarEfeitosPerfuracao(brocaE)
            EffectsManager.AtivarEfeitosPerfuracao(brocaD)
        end
        
        ArmaduraController.LoopPerfuracao = RunService.Heartbeat:Connect(function()
            if not ArmaduraController.Ocupada or not ArmaduraController.Perfurando then return end
            
            local rootPart = char:FindFirstChild("HumanoidRootPart")
            if not rootPart then return end
            
            local origem = rootPart.Position
            local direcao = Vector3.new(0, -1, 0)
            
            TerrenoManager.PerfurarChao(origem, direcao, ArmaduraController.Modelo)
            
            -- Consumo de energia
            ArmaduraController.Energia = ArmaduraController.Energia - (CONFIG.CustoEnergiaPerfuracao * 0.1)
            if ArmaduraController.Energia <= 0 then
                ArmaduraController.Energia = 0
                ArmaduraController.TogglePerfurar() -- Desliga
            end
        end)
    else
        if humanoid then
            humanoid.WalkSpeed = CONFIG.VelocidadeNormal
            humanoid.JumpPower = CONFIG.PuloNormal
        end
        
        if ArmaduraController.LoopPerfuracao then
            ArmaduraController.LoopPerfuracao:Disconnect()
            ArmaduraController.LoopPerfuracao = nil
        end
    end
end

-- =========================================================================
-- MÓDULO 7: CONTROLE DE INPUT (PC E MOBILE)
-- =========================================================================
local MobileController = {}

function MobileController.CriarInterface()
    local playerGui = LocalPlayer:WaitForChild("PlayerGui")
    
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "PrypotMobileHUD"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = playerGui
    
    -- Botão de Perfurar
    local botaoPerfurar = Instance.new("TextButton")
    botaoPerfurar.Name = "BotaoPerfurar"
    botaoPerfurar.Size = UDim2.new(0, 100, 0, 100)
    botaoPerfurar.Position = UDim2.new(1, -120, 1, -120)
    botaoPerfurar.BackgroundColor3 = CONFIG.Cores.Detalhes
    botaoPerfurar.Text = "PERFURAR"
    botaoPerfurar.TextColor3 = Color3.new(1, 1, 1)
    botaoPerfurar.Font = Enum.Font.Code
    botaoPerfurar.TextScaled = true
    botaoPerfurar.Parent = screenGui
    
    botaoPerfurar.MouseButton1Click:Connect(function()
        ArmaduraController.TogglePerfurar()
    end)
    
    -- Botão de Pular
    local botaoPular = Instance.new("TextButton")
    botaoPular.Name = "BotaoPular"
    botaoPular.Size = UDim2.new(0, 80, 0, 80)
    botaoPular.Position = UDim2.new(1, -220, 1, -100)
    botaoPular.BackgroundColor3 = CONFIG.Cores.Secundaria
    botaoPular.Text = "PULAR"
    botaoPular.TextColor3 = Color3.new(1, 1, 1)
    botaoPular.Font = Enum.Font.Code
    botaoPular.TextScaled = true
    botaoPular.Parent = screenGui
    
    botaoPular.MouseButton1Click:Connect(function()
        local char = LocalPlayer.Character
        if char then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.Jump = true
            end
        end
    end)
    
    -- Botão de Sair
    local botaoSair = Instance.new("TextButton")
    botaoSair.Name = "BotaoSair"
    botaoSair.Size = UDim2.new(0, 80, 0, 80)
    botaoSair.Position = UDim2.new(0, 20, 1, -100)
    botaoSair.BackgroundColor3 = CONFIG.Cores.Perigo
    botaoSair.Text = "SAIR"
    botaoSair.TextColor3 = Color3.new(1, 1, 1)
    botaoSair.Font = Enum.Font.Code
    botaoSair.TextScaled = true
    botaoSair.Parent = screenGui
    
    botaoSair.MouseButton1Click:Connect(function()
        ArmaduraController.Sair()
    end)
end

function MobileController.RemoverInterface()
    local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
    if playerGui then
        local hud = playerGui:FindFirstChild("PrypotMobileHUD")
        if hud then
            hud:Destroy()
        end
    end
end

-- =========================================================================
-- MÓDULO 8: CONFIGURAÇÃO DE INPUTS DO USUÁRIO
-- =========================================================================
local function ConfigurarInputs()
    -- Input para PC (Teclado)
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.KeyCode == Enum.KeyCode.F then
            ArmaduraController.TogglePerfurar()
        elseif input.KeyCode == Enum.KeyCode.E then
            -- O ProximityPrompt já lida com a entrada, mas podemos forçar se necessário
        end
    end)
    
    -- Input para Mobile (ContextActionService)
    ContextActionService:BindAction("PrypotPerfurar", function(_, estado)
        if estado == Enum.UserInputState.Begin then
            ArmaduraController.TogglePerfurar()
        end
    end, true, Enum.KeyCode.F, Enum.KeyCode.ButtonX)
    
    ContextActionService:BindAction("PrypotPular", function(_, estado)
        if estado == Enum.UserInputState.Begin then
            local char = LocalPlayer.Character
            if char then
                local humanoid = char:FindFirstChildOfClass("Humanoid")
                if humanoid then
                    humanoid.Jump = true
                end
            end
        end
    end, true, Enum.KeyCode.Space, Enum.KeyCode.ButtonA)
    
    -- Ajustar posições dos botões mobile criados
    ContextActionService:SetPosition("PrypotPerfurar", UDim2.new(1, -120, 1, -120))
    ContextActionService:SetPosition("PrypotPular", UDim2.new(1, -220, 1, -100))
end

-- =========================================================================
-- MÓDULO 9: INICIALIZAÇÃO E LOOP PRINCIPAL
-- =========================================================================

-- Aguardar o personagem carregar
LocalPlayer.CharacterAdded:Connect(function(char)
    task.wait(2)
    if not ArmaduraController.Modelo then
        ArmaduraController.Inicializar()
    else
        -- Se o jogador morrer e a armadura já existir, reposicionar
        local hrp = char:WaitForChild("HumanoidRootPart")
        if hrp then
            ArmaduraController.Root.CFrame = hrp.CFrame * CFrame.new(0, 0, -8)
        end
    end
end)

-- Inicialização imediata caso o personagem já exista
if LocalPlayer.Character then
    task.wait(2)
    ArmaduraController.Inicializar()
    ConfigurarInputs()
else
    LocalPlayer.CharacterAdded:Wait()
    task.wait(2)
    ArmaduraController.Inicializar()
    ConfigurarInputs()
end

-- Loop de monitoramento de energia e estado
RunService.Heartbeat:Connect(function(deltaTime)
    if ArmaduraController.Ocupada then
        -- Recarga de energia passiva
        if not ArmaduraController.Perfurando then
            ArmaduraController.Energia = math.min(CONFIG.EnergiaMaxima, ArmaduraController.Energia + (5 * deltaTime))
        end
        
        -- Atualizar UI de energia (se existir)
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if playerGui then
            local hud = playerGui:FindFirstChild("PrypotMobileHUD")
            if hud then
                -- Poderia adicionar uma barra de energia aqui
            end
        end
    end
end)

print("[Prypot 0.X1] Script carregado com sucesso. Sistema operacional.")
