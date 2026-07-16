// ==UserScript==
// @name         Openfront 3x2 Lobby Grid
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  Autoloads the 3x2 lobby grid layout on openfront.io
// @author       blon
// @match        *://openfront.io/*
// @match        *://*.openfront.io/*
// @allFrames    true
// @grant        none
// @run-at       document-start
// @license      MIT
// ==/UserScript==

(function() {
  "use strict";

  let lobbyWS = null;
  let lobbyReconnectTimeout = null;
  let lobbyShouldReconnect = true;
  let lobbyConnected = false;
  let lobbyLatestPayload = null;

  function getNumWorkers() {
    const bc = window.BOOTSTRAP_CONFIG;
    return bc && Number.isInteger(bc.numWorkers) && bc.numWorkers > 0 ? bc.numWorkers : 8;
  }

  function simpleHashForWorkerPath(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = (hash << 5) - hash + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash);
  }

  function getCurrentWorkerPath() {
    const pathParts = window.location.pathname.split("/").filter(Boolean);
    const candidate = pathParts[0];
    return /^w\d+$/.test(candidate) ? candidate : null;
  }

  function getWorkerPath(gameID) {
    const currentPath = getCurrentWorkerPath();
    if (currentPath) {
      return currentPath;
    }
    return `w${simpleHashForWorkerPath(gameID) % getNumWorkers()}`;
  }

  function getLobbyUrl(gameID) {
    const path = getWorkerPath(gameID);
    return `${window.location.origin}/${path}/game/${encodeURIComponent(gameID)}`;
  }

  function formatDurationCard(ms) {
    if (ms <= 0) return "Now";
    const totalSec = Math.ceil(ms / 1e3);
    const minutes = Math.floor(totalSec / 60);
    const seconds = totalSec % 60;
    if (minutes > 0 && seconds > 0) return `${minutes}min ${seconds}s`;
    if (minutes > 0) return `${minutes}min`;
    return `${seconds}s`;
  }

  function getLobbyMapSlug(mapName) {
    if (typeof mapName !== "string") return "";
    return mapName.toLowerCase().replace(/[\s_]/g, "").replace(/[^\w]/g, "");
  }

  function getLobbyMapThumbnailUrl(mapName) {
    const slug = getLobbyMapSlug(mapName);
    if (!slug) return "";
    return `https://raw.githubusercontent.com/openfrontio/OpenFrontIO/main/resources/maps/${slug}/thumbnail.webp`;
  }

  function getLobbyModifiers(game) {
    const pm = game.publicGameModifiers || {};
    const gcfg = game.gameConfig || {};
    const disabled = Array.isArray(gcfg.disabledUnits) ? gcfg.disabledUnits : [];
    const labels = [];
    const goldMult = game.goldMultiplier ?? gcfg.goldMultiplier;
    const goldMultN = typeof goldMult === "number" ? goldMult : parseFloat(goldMult);
    if (Number.isFinite(goldMultN) && goldMultN !== 1) labels.push(`x${goldMultN} Gold Multiplier`);
    const startGold = game.startingGold ?? gcfg.startingGold;
    const startGoldN = typeof startGold === "number" ? startGold : parseFloat(startGold);
    if (Number.isFinite(startGoldN) && startGoldN > 0) {
      const display = startGoldN >= 1e6 ? `${startGoldN / 1e6}M` : startGoldN >= 1e3 ? `${startGoldN / 1e3}K` : String(startGoldN);
      labels.push(`${display} Starting Gold`);
    }
    if (pm.isRandomSpawn === true || gcfg.randomSpawn === true) labels.push("Random Spawn");
    if (pm.isCompact === true || gcfg.gameMapSize === "Compact") labels.push("Compact Map");
    if (pm.isCrowded === true) labels.push("Crowded");
    if (pm.isHardNations === true) labels.push("Hard Nations");
    if (pm.isAlliancesDisabled === true || gcfg.disableAlliances === true) labels.push("Alliances Disabled");
    if (pm.isPortsDisabled === true || disabled.includes("Port")) labels.push("Ports Disabled");
    if (pm.isNukesDisabled === true || disabled.includes("Atom Bomb")) labels.push("Nukes Disabled");
    if (pm.isSAMsDisabled === true) labels.push("SAMs Disabled");
    if (pm.isPeaceTime === true) labels.push("4min Peace");
    if (pm.isWaterNukes === true || gcfg.waterNukes === true) labels.push("Water Nukes");
    return labels;
  }

  function getLobbyModeLabel(game) {
    const cfg2 = game.gameConfig;
    const mode = cfg2 && typeof cfg2.gameMode === "string" ? cfg2.gameMode : null;
    const totalPlayers = cfg2?.maxPlayers ?? game.numClients ?? void 0;
    if (mode === "Free For All") {
      return "FFA";
    }
    if (mode === "Team") {
      if (cfg2?.playerTeams === "Humans Vs Nations") {
        return totalPlayers ? `Humans vs Nations (${totalPlayers} players)` : "Humans vs Nations";
      }
      const namedTeamSizes = {
        Duos: 2,
        Trios: 3,
        Quads: 4
      };
      if (typeof cfg2?.playerTeams === "string" && namedTeamSizes[cfg2.playerTeams]) {
        const playersPerTeam = namedTeamSizes[cfg2.playerTeams];
        if (totalPlayers) {
          const teamCount = Math.floor(totalPlayers / playersPerTeam);
          return `${cfg2.playerTeams} (${teamCount} teams of ${playersPerTeam})`;
        }
        return cfg2.playerTeams;
      }
      if (typeof cfg2?.playerTeams === "number" && cfg2.playerTeams > 0) {
        const teamCount = cfg2.playerTeams;
        if (totalPlayers) {
          const teamCountN = teamCount;
          const playersPerTeam = Math.floor(totalPlayers / teamCountN);
          return `${teamCountN} teams of ${playersPerTeam}`;
        }
        return `${teamCount} teams`;
      }
      return "Team";
    }
    if (game.publicGameType) {
      return String(game.publicGameType).toUpperCase();
    }
    return "UNKNOWN";
  }

  function getLobbyMapDisplay(game) {
    try {
      if (!game || typeof game !== "object") return "Unknown";
      const resolveVal = (val) => {
        if (val == null) return null;
        if (typeof val === "string" && val.trim()) return val.trim();
        if (typeof val === "number") return String(val);
        if (typeof val === "object") {
          if (typeof val.name === "string" && val.name.trim()) return val.name.trim();
          if (typeof val.mapName === "string" && val.mapName.trim()) return val.mapName.trim();
          if (typeof val.gameMap === "string" && val.gameMap.trim()) return val.gameMap.trim();
          if (typeof val.id === "string" && val.id.trim()) return val.id.trim();
          if (typeof val.id === "number") return String(val.id);
        }
        return null;
      };
      const topCandidates = [
        "gameMap",
        "game_map",
        "map",
        "mapName",
        "map_name",
        "terrain",
        "terrainName",
        "terrain_name",
        "mapId",
        "map_id"
      ];
      for (const k of topCandidates) {
        if (Object.prototype.hasOwnProperty.call(game, k)) {
          const v = resolveVal(game[k]);
          if (v) return v;
        }
      }
      if (game.gameConfig && typeof game.gameConfig === "object") {
        for (const k of topCandidates) {
          if (Object.prototype.hasOwnProperty.call(game.gameConfig, k)) {
            const v = resolveVal(game.gameConfig[k]);
            if (v) return v;
          }
        }
        if (Object.prototype.hasOwnProperty.call(game.gameConfig, "gameMap")) {
          const v = resolveVal(game.gameConfig.gameMap);
          if (v) return v;
        }
      }
      for (const key of Object.keys(game)) {
        if (/map|terrain/i.test(key)) {
          const v = resolveVal(game[key]);
          if (v) return v;
        }
      }
    } catch (e) {
    }
    return "Unknown";
  }

  function getAccessibleDocuments() {
    const docs = [document];
    try {
      for (const frame of document.querySelectorAll("iframe")) {
        try {
          if (frame.contentDocument) docs.push(frame.contentDocument);
        } catch (e) {
        }
      }
    } catch (e) {
    }
    return docs;
  }

  function querySelectorAllAcrossDocs(selector) {
    const els = [];
    for (const doc of getAccessibleDocuments()) {
      try {
        const list = doc.querySelectorAll(selector);
        for (const el of list) {
          els.push(el);
        }
      } catch (e) {
      }
    }
    return els;
  }

  function querySelectorAcrossDocs(selector) {
    for (const doc of getAccessibleDocuments()) {
      try {
        const el = doc.querySelector(selector);
        if (el) return el;
      } catch (e) {
      }
    }
    return null;
  }

  function makeUpcomingLobbyCard(game, serverTime) {
    const mapName = getLobbyMapDisplay(game);
    const thumbUrl = getLobbyMapThumbnailUrl(mapName);
    const modeLabel = getLobbyModeLabel(game);
    const modifiers = getLobbyModifiers(game);
    const maxPlayers = game.gameConfig?.maxPlayers ?? game.numClients ?? "?";
    const numClients = game.numClients ?? 0;
    const startAt = typeof game.startsAt === "number" ? game.startsAt : serverTime;
    const timeDelta = startAt - serverTime;
    const isLive = timeDelta <= 0;
    const timerText = isLive ? "Open" : formatDurationCard(timeDelta);
    const gameID = game.gameID;
    const badgesHtml = modifiers.slice(0, 2).map(
      (m) => `<span class="px-2 py-1 rounded text-xs font-bold uppercase tracking-widest bg-malibu-blue text-white shadow-[var(--shadow-malibu-blue-pill)]">${m}</span>`
    ).join("");
    const modifiersHtml = modifiers.length > 0 ? `<div class="flex flex-col items-start gap-1 mt-[2px]">${badgesHtml}</div>` : `<div></div>`;
    const imgHtml = thumbUrl ? `<img draggable="false" src="${thumbUrl}" alt="${mapName}" class="absolute inset-0 w-full h-full object-cover object-center scale-[1.05] [image-rendering:auto]">` : "";
    return { gameID, html: `
      <button class="group relative w-full h-44 sm:h-full text-white uppercase rounded-2xl transition-all duration-200 hover:scale-[1.02] active:scale-[0.98] bg-surface hover:shadow-[var(--shadow-lobby-card-hover)] blon-upcoming-card">
        <div class="absolute inset-0 rounded-2xl overflow-hidden pointer-events-none">
          ${imgHtml}
        </div>
        <div class="absolute inset-x-2 top-2 flex items-start justify-between gap-2">
          ${modifiersHtml}
          ${!isLive ? `<div class="shrink-0"><span class="text-xs font-bold tracking-widest normal-case bg-malibu-blue text-white px-2 py-1 rounded">${timerText}</span></div>` : "<div></div>"}
        </div>
        <div class="absolute bottom-0 left-0 right-0 flex flex-col px-3 py-2 bg-black/55 backdrop-blur-sm rounded-b-2xl" style="overflow:visible;">
          <div class="absolute bottom-full right-2 mb-1 flex items-center gap-1.5 z-10">
            <div class="blon-copy-btn flex items-center gap-1 text-[10px] font-bold tracking-widest bg-neutral-800 hover:bg-neutral-700 text-neutral-300 hover:text-white backdrop-blur-sm px-2 py-0.5 rounded transition-colors cursor-pointer select-none normal-case" title="Copy Invite Link" data-copy-game-id="${gameID}">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-3 w-3" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
                  <path stroke-linecap="round" stroke-linejoin="round" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
              </svg>
              <span>Copy</span>
            </div>
            <span class="flex items-center gap-1 text-xs font-bold tracking-widest bg-black/70 backdrop-blur-sm px-2 py-0.5 rounded">
              ${numClients}/${maxPlayers}
              <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 inline-block" viewBox="0 0 20 20" fill="currentColor">
                <path d="M13 6a3 3 0 11-6 0 3 3 0 016 0zM18 8a2 2 0 11-4 0 2 2 0 014 0zM14 15a4 4 0 00-8 0v3h8v-3zM6 8a2 2 0 11-4 0 2 2 0 014 0zM16 18v-3a5.972 5.972 0 00-.75-2.906A3.005 3.005 0 0119 15v3h-3zM4.75 12.094A5.973 5.973 0 004 15v3H1v-3a3 3 0 013.75-2.906z"></path>
              </svg>
            </span>
          </div>
          <p class="text-sm sm:text-base font-bold uppercase tracking-wider text-left leading-tight">${mapName}</p>
          <h3 class="text-xs text-white/70 uppercase tracking-wider text-left">${modeLabel}</h3>
        </div>
      </button>
    ` };
  }

  function updateLobbyGridLayout() {
    let styleEl = document.getElementById("blon-lobby-3x2-grid-css");
    if (!styleEl) {
      styleEl = document.createElement("style");
      styleEl.id = "blon-lobby-3x2-grid-css";
      styleEl.textContent = `
          @media (min-width: 640px) {
              div.grid[class*="sm:grid-cols-[2fr_1fr]"] {
                  grid-template-columns: repeat(3, minmax(0, 1fr)) !important;
                  height: auto !important;
              }
              div.grid[class*="sm:grid-cols-[2fr_1fr]"] > div[class*="sm:block"],
              div.grid[class*="sm:grid-cols-[2fr_1fr]"] > div[class*="sm:flex"] {
                  display: none !important;
              }
              div.grid[class*="sm:grid-cols-[2fr_1fr]"] > div[class*="sm:hidden"] {
                  display: block !important;
              }
              div.grid[class*="sm:grid-cols-[2fr_1fr]"] > div[class*="sm:hidden"] button {
                  height: 12rem !important;
              }
              .blon-upcoming-slot {
                  display: block !important;
              }
              .blon-upcoming-slot button {
                  height: 12rem !important;
              }
          }
      `;
      (document.head || document.documentElement).appendChild(styleEl);
    }

    if (!lobbyLatestPayload || !lobbyLatestPayload.games) return;
    const serverTime = typeof lobbyLatestPayload.serverTime === "number" ? lobbyLatestPayload.serverTime : Date.now();
    const allGames = Object.values(lobbyLatestPayload.games).flat();

    const addCopyButtonToRealCard = (el, gameID) => {
      const button = el.querySelector("button");
      if (!button) return;
      const bottomBar = Array.from(button.children).find((child) => {
        const className = child.className || "";
        return className.includes("bottom-0") && className.includes("bg-black");
      });
      if (!bottomBar) return;
      if (bottomBar.querySelector(".blon-copy-btn")) return;
      let playerSpan = bottomBar.querySelector("span.absolute.bottom-full.right-2");
      let parentDiv = bottomBar.querySelector("div.absolute.bottom-full.right-2");
      if (!playerSpan) {
        const playerSvg = bottomBar.querySelector('svg[viewBox="0 0 20 20"]');
        if (playerSvg) {
          playerSpan = playerSvg.closest("span");
        }
      }
      if (!playerSpan) return;
      if (!parentDiv) {
        parentDiv = playerSpan.parentElement;
        if (!parentDiv || !parentDiv.classList.contains("bottom-full")) {
          parentDiv = null;
        }
      }
      const copyBtn = document.createElement("div");
      copyBtn.className = "blon-copy-btn flex items-center gap-1 text-[10px] font-bold tracking-widest bg-neutral-800 hover:bg-neutral-700 text-neutral-300 hover:text-white backdrop-blur-sm px-2 py-0.5 rounded transition-colors cursor-pointer select-none normal-case";
      copyBtn.title = "Copy Invite Link";
      copyBtn.dataset.copyGameId = String(gameID);
      copyBtn.innerHTML = `
          <svg xmlns="http://www.w3.org/2000/svg" class="h-3 w-3" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5">
              <path stroke-linecap="round" stroke-linejoin="round" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
          </svg>
          <span>Copy</span>
      `;
      copyBtn.addEventListener("click", (e) => {
        e.stopPropagation();
        e.preventDefault();
        const url = getLobbyUrl(gameID);
        navigator.clipboard.writeText(url).then(() => {
          const label = copyBtn.querySelector("span");
          if (label) {
            const orig = label.textContent;
            label.textContent = "Copied!";
            copyBtn.style.color = "#00ff66";
            copyBtn.style.borderColor = "#00ff66";
            setTimeout(() => {
              label.textContent = orig;
              copyBtn.style.color = "";
              copyBtn.style.borderColor = "";
            }, 1500);
          }
        }).catch((err) => {
        });
      });
      if (parentDiv && parentDiv.classList.contains("flex")) {
        parentDiv.insertBefore(copyBtn, playerSpan);
      } else {
        const wrapper = document.createElement("div");
        wrapper.className = "absolute bottom-full right-2 mb-1 flex items-center gap-1.5 z-10";
        playerSpan.classList.remove("absolute", "bottom-full", "right-2", "mb-1");
        playerSpan.parentNode.insertBefore(wrapper, playerSpan);
        wrapper.appendChild(copyBtn);
        wrapper.appendChild(playerSpan);
      }
    };

    const ffaGame = allGames.find((g) => g.gameConfig?.gameMode === "Free For All");
    const teamsGame = allGames.find((g) => g.gameConfig?.gameMode === "Team");
    const specialGame = allGames.find((g) => g.gameConfig?.gameMode !== "Free For All" && g.gameConfig?.gameMode !== "Team");

    querySelectorAllAcrossDocs('div.grid[class*="sm:grid-cols-[2fr_1fr]"]').forEach((grid) => {
      const realCards = Array.from(grid.children).filter((el) => el.classList.contains("sm:hidden") || el.className.includes("sm:hidden"));
      realCards.forEach((el, index) => {
        const btn = el.querySelector("button");
        if (!btn) return;
        let matchedGame = null;
        if (index === 0) matchedGame = specialGame;
        else if (index === 1) matchedGame = ffaGame;
        else if (index === 2) matchedGame = teamsGame;
        if (!matchedGame) {
          const p = el.querySelector("p");
          const h3 = el.querySelector("h3");
          if (p && h3) {
            const mapText = p.textContent.trim().toLowerCase();
            const modeText = h3.textContent.trim().toLowerCase();
            const normalizeMode = (m) => m === "ffa" ? "free for all" : m;
            matchedGame = allGames.find((g) => {
              const gMap = getLobbyMapDisplay(g).toLowerCase();
              const gMode = getLobbyModeLabel(g).toLowerCase();
              return gMap === mapText && normalizeMode(gMode) === normalizeMode(modeText);
            });
          }
        }
        if (matchedGame) {
          btn.dataset.blonGameid = String(matchedGame.gameID);
          addCopyButtonToRealCard(el, matchedGame.gameID);
        }
      });

      const existingFillers = Array.from(grid.querySelectorAll(":scope > .blon-upcoming-slot"));
      const fillerCount = Math.max(0, 6 - realCards.length);
      const shownIds = new Set(
        realCards.map((el) => {
          const btn = el.querySelector("button[data-blon-gameid]");
          return btn ? btn.dataset.blonGameid : null;
        }).filter(Boolean)
      );
      const candidates = allGames.filter((g) => !shownIds.has(String(g.gameID))).sort((a, b) => {
        const at = typeof a.startsAt === "number" ? a.startsAt : serverTime;
        const bt = typeof b.startsAt === "number" ? b.startsAt : serverTime;
        return at - bt;
      }).slice(0, fillerCount);

      const currentFillerIds = existingFillers.map(el => {
        const btn = el.querySelector("button[data-blon-gameid]");
        return btn ? btn.dataset.blonGameid : null;
      }).filter(Boolean);
      const candidateIds = candidates.map(g => String(g.gameID));

      const isSame = currentFillerIds.length === candidateIds.length && currentFillerIds.every((id, idx) => id === candidateIds[idx]);
      if (!isSame) {
        existingFillers.forEach((el) => el.remove());
        candidates.forEach((game) => {
          const { gameID, html } = makeUpcomingLobbyCard(game, serverTime);
          const wrapper = document.createElement("div");
          wrapper.className = "blon-upcoming-slot";
          wrapper.innerHTML = html;
          const btn = wrapper.querySelector("button");
          if (btn) {
            btn.dataset.blonGameid = String(gameID);
            btn.addEventListener("click", () => {
              const lobbyId = gameID;
              try {
                if (typeof window.showPage === "function") window.showPage("page-join-lobby");
                document.dispatchEvent(new CustomEvent("join-lobby", {
                  detail: { gameID: lobbyId, source: "public" },
                  bubbles: true,
                  composed: true
                }));
                return;
              } catch (e) {
              }
              const joinModal = querySelectorAcrossDocs("join-lobby-modal");
              if (joinModal && typeof joinModal.open === "function") {
                try {
                  joinModal.open({ lobbyId });
                  return;
                } catch (e) {
                }
              }
              window.location.href = getLobbyUrl(lobbyId);
            });
          }
          const copyBtn = wrapper.querySelector(".blon-copy-btn");
          if (copyBtn) {
            copyBtn.addEventListener("click", (e) => {
              e.stopPropagation();
              e.preventDefault();
              const url = getLobbyUrl(gameID);
              navigator.clipboard.writeText(url).then(() => {
                const label = copyBtn.querySelector("span");
                if (label) {
                  const orig = label.textContent;
                  label.textContent = "Copied!";
                  copyBtn.style.color = "#00ff66";
                  copyBtn.style.borderColor = "#00ff66";
                  setTimeout(() => {
                    label.textContent = orig;
                    copyBtn.style.color = "";
                    copyBtn.style.borderColor = "";
                  }, 1500);
                }
              }).catch((err) => {
              });
            });
          }
          grid.appendChild(wrapper);
        });
      }
    });
  }

  function handleLobbySocketMessage(event) {
    try {
      const payload = JSON.parse(event.data);
      if (payload && typeof payload.serverTime === "number" && payload.games) {
        lobbyLatestPayload = payload;
        updateLobbyGridLayout();
      }
    } catch (e) {
    }
  }

  function connectLobbyFeed() {
    if (lobbyWS) return;
    lobbyShouldReconnect = true;
    const workerPath = `/w${Math.floor(Math.random() * getNumWorkers())}`;
    const protocol = window.location.protocol === "https:" ? "wss:" : "ws:";
    const url = `${protocol}//${window.location.host}${workerPath}/lobbies`;
    try {
      lobbyWS = new window.WebSocket(url);
      lobbyWS.addEventListener("open", () => {
        lobbyConnected = true;
        if (lobbyReconnectTimeout) {
          clearTimeout(lobbyReconnectTimeout);
          lobbyReconnectTimeout = null;
        }
        updateLobbyGridLayout();
      });
      lobbyWS.addEventListener("message", handleLobbySocketMessage);
      lobbyWS.addEventListener("close", () => {
        lobbyConnected = false;
        lobbyWS = null;
        scheduleLobbyReconnect();
      });
      lobbyWS.addEventListener("error", () => {
        lobbyConnected = false;
        scheduleLobbyReconnect();
      });
    } catch (e) {
      lobbyWS = null;
      scheduleLobbyReconnect();
    }
  }

  function scheduleLobbyReconnect() {
    if (!lobbyShouldReconnect || lobbyReconnectTimeout) return;
    lobbyReconnectTimeout = window.setTimeout(() => {
      lobbyReconnectTimeout = null;
      if (lobbyShouldReconnect) connectLobbyFeed();
    }, 1000);
  }

  connectLobbyFeed();

  setInterval(() => {
    const grids = querySelectorAllAcrossDocs('div.grid[class*="sm:grid-cols-[2fr_1fr]"]');
    if (grids.length > 0) {
      const styleEl = document.getElementById("blon-lobby-3x2-grid-css");
      const fillers = querySelectorAllAcrossDocs(".blon-upcoming-slot");
      if (!styleEl || fillers.length === 0) {
        updateLobbyGridLayout();
      }
    }
  }, 500);

})();
