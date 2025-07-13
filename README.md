# aplicabilidad-juego
import { useState, useEffect, useRef } from "react";

export default function App() {
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  const [passwordInput, setPasswordInput] = useState("");
  const [appPassword, setAppPassword] = useState(() => {
    return localStorage.getItem("appPassword") || "1234";
  });
  const sportsBettingBlacklist = [
    "bet", "apuesta", "deportes", "sport", "bwin", "888sport", "marathon",
    "williamhill", "pinnacle", "1xbet", "bet365", "ladbrokes", "coral",
    "netbet", "codere", ".bet", ".apuestas", ".sports"
  ];
  const [appList, setAppList] = useState(() => {
    const saved = localStorage.getItem("appList");
    return saved ? JSON.parse(saved) : [
      { id: 1, name: "Instagram", limit: 30, used: 15 },
      { id: 2, name: "Facebook", limit: 45, used: 50 },
      { id: 3, name: "TikTok", limit: 60, used: 60 },
      { id: 4, name: "Twitter", limit: 20, used: 10 },
    ];
  });
  const [newAppName, setNewAppName] = useState("");
  const [newAppLimit, setNewAppLimit] = useState(0);
  const [isAddingApp, setIsAddingApp] = useState(false);
  const [notification, setNotification] = useState("");
  const [isConfirmingDelete, setIsConfirmingDelete] = useState(null);
  const [deletePassword, setDeletePassword] = useState("");
  const [showChangePassword, setShowChangePassword] = useState(false);
  const [newPassword, setNewPassword] = useState("");
  const [searchTerm, setSearchTerm] = useState("");
  const [isExtensionActive, setIsExtensionActive] = useState(false);
  const canvasRef = useRef(null);

  // Simular uso de tiempo incrementando aleatoriamente cada minuto
  useEffect(() => {
    const interval = setInterval(() => {
      setAppList((prev) =>
        prev.map((app) => {
          if (Math.random() < 0.3) {
            const newUsed = Math.min(app.used + 1, app.limit + 10);
            if (newUsed > app.limit && app.used <= app.limit) {
              showNotification(`¡Has superado el límite en ${app.name}!`);
            }
            return { ...app, used: newUsed };
          }
          return app;
        })
      );
    }, 60000); // 1 minuto real = 1 minuto simulado
    return () => clearInterval(interval);
  }, []);

  // Reiniciar uso diario a medianoche
  useEffect(() => {
    const now = new Date();
    const midnight = new Date(now);
    midnight.setHours(24, 0, 0, 0);
    const timeUntilMidnight = midnight.getTime() - now.getTime();
    const timeout = setTimeout(() => {
      resetDailyUsage();
      const nextDayTimeout = setInterval(resetDailyUsage, 86400000); // cada 24 horas
      return () => clearInterval(nextDayTimeout);
    }, timeUntilMidnight);
    return () => clearTimeout(timeout);
  }, []);

  // Guardar cambios en localStorage
  useEffect(() => {
    localStorage.setItem("appList", JSON.stringify(appList));
  }, [appList]);

  // Verificar si la extensión está activa
  useEffect(() => {
    if (window.chrome && chrome.runtime) {
      chrome.runtime.connect({ name: "popup" });
      chrome.runtime.sendMessage({ action: "checkStatus" }, (response) => {
        setIsExtensionActive(response?.blockingEnabled || false);
      });
    }
  }, []);

  // Limpiar notificación después de 5 segundos
  useEffect(() => {
    if (notification) {
      const timer = setTimeout(() => {
        setNotification("");
      }, 5000);
      return () => clearTimeout(timer);
    }
  }, [notification]);

  // Dibujar gráfico de barras
  useEffect(() => {
    const ctx = canvasRef.current?.getContext("2d");
    if (!ctx) return;
    ctx.clearRect(0, 0, canvasRef.current.width, canvasRef.current.height);
    const barWidth = 40;
    const spacing = 20;
    const maxUsed = Math.max(...appList.map((a) => a.used), 1);
    const maxHeight = 200;

    appList.forEach((app, i) => {
      const barHeight = (app.used / maxUsed) * maxHeight;
      const x = i * (barWidth + spacing);
      const y = maxHeight - barHeight;
      ctx.fillStyle = "#6366F1";
      ctx.fillRect(x, y, barWidth, barHeight);
      ctx.fillStyle = "#000";
      ctx.font = "10px sans-serif";
      ctx.fillText(app.name, x, maxHeight + 15);
    });
  }, [appList]);

  const handleLogin = () => {
    if (passwordInput === appPassword) {
      setIsLoggedIn(true);
    } else {
      alert("Contraseña incorrecta");
    }
  };

  const isBlockedApp = (name) => {
    const lowerName = name.toLowerCase();
    return sportsBettingBlacklist.some((keyword) => lowerName.includes(keyword.toLowerCase()));
  };

  const handleAddApp = () => {
    if (!newAppName || newAppLimit <= 0) return;
    if (isBlockedApp(newAppName)) {
      alert("Esta aplicación contiene palabras prohibidas (apuestas deportivas). No puedes añadirla.");
      return;
    }
    const newApp = {
      id: Date.now(),
      name: newAppName,
      limit: parseInt(newAppLimit),
      used: 0,
    };
    setAppList([...appList, newApp]);
    setNewAppName("");
    setNewAppLimit(0);
    setIsAddingApp(false);
  };

  const deleteApp = () => {
    if (deletePassword !== appPassword) {
      alert("Contraseña incorrecta");
      return;
    }
    setAppList(appList.filter((app) => app.id !== isConfirmingDelete));
    setIsConfirmingDelete(null);
    setDeletePassword("");
  };

  const changePassword = () => {
    if (!newPassword.trim()) {
      alert("La nueva contraseña no puede estar vacía.");
      return;
    }
    setAppPassword(newPassword);
    localStorage.setItem("appPassword", newPassword);
    setShowChangePassword(false);
    setNewPassword("");
  };

  const resetUsage = (id) => {
    setAppList(
      appList.map((app) =>
        app.id === id ? { ...app, used: 0 } : app
      )
    );
  };

  const resetDailyUsage = () => {
    setAppList(
      appList.map((app) => ({ ...app, used: 0 }))
    );
  };

  const formatTime = (minutes) => {
    const hours = Math.floor(minutes / 60);
    const mins = minutes % 60;
    return `${hours}h ${mins}m`;
  };

  const showNotification = (message) => {
    setNotification(message);
  };

  const exportToCSV = () => {
    const csvRows = [];
    csvRows.push(["Nombre", "Tiempo Usado (min)", "Límite (min)"].join(","));
    appList.forEach((app) => {
      csvRows.push([app.name, app.used, app.limit].join(","));
    });
    const csvString = csvRows.join("\n");
    const blob = new Blob([csvString], { type: "text/csv" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "uso_aplicaciones.csv";
    link.click();
    URL.revokeObjectURL(url);
  };

  const filteredApps = appList.filter((app) =>
    app.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  if (!isLoggedIn) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-6">
        <div className="bg-white rounded-xl shadow-lg p-8 w-full max-w-md">
          <h2 className="text-2xl font-bold text-indigo-700 mb-6 text-center">Iniciar Sesión</h2>
          <input
            type="password"
            value={passwordInput}
            onChange={(e) => setPasswordInput(e.target.value)}
            placeholder="Ingresa tu contraseña"
            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-300 mb-4"
          />
          <button
            onClick={handleLogin}
            className="w-full bg-indigo-600 hover:bg-indigo-700 text-white py-2 rounded-lg transition"
          >
            Acceder
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-6">
      <div className="max-w-4xl mx-auto">
        {/* Notificación */}
        {notification && (
          <div className="fixed top-4 right-4 bg-red-500 text-white px-6 py-3 rounded shadow-lg z-50 animate-fade-in-down">
            {notification}
          </div>
        )}
        <header className="text-center mb-10">
          <h1 className="text-4xl font-bold text-indigo-700 mb-2">Control de Uso de Aplicaciones</h1>
          <p className="text-gray-600">Administra tu tiempo de uso de aplicaciones diariamente</p>
        </header>
        <div className="flex justify-between items-center mb-4">
          <input
            type="text"
            placeholder="Buscar aplicación..."
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            className="px-4 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-indigo-300"
          />
          <div className="flex space-x-2">
            <button
              onClick={() => setShowChangePassword(true)}
              className="text-sm text-indigo-600 hover:text-indigo-800 underline"
            >
              Cambiar contraseña
            </button>
            <button
              onClick={exportToCSV}
              className="text-sm text-green-600 hover:text-green-800 underline"
            >
              Exportar a CSV
            </button>
            <button
              onClick={() => window.open(chrome.runtime.getURL("popup.html"), "_blank")}
              className={`text-sm ${isExtensionActive ? "text-red-600" : "text-gray-600"} underline`}
            >
              {isExtensionActive ? "Bloqueo Activo" : "Activar Bloqueo"}
            </button>
          </div>
        </div>
        <div className="bg-white rounded-xl shadow-lg p-6 mb-8">
          <h2 className="text-2xl font-semibold text-gray-800 mb-4">Gráfico de Uso</h2>
          <canvas ref={canvasRef} width="600" height="250" className="mb-6"></canvas>
        </div>
        <div className="bg-white rounded-xl shadow-lg p-6 mb-8">
          <div className="flex justify-between items-center mb-4">
            <h2 className="text-2xl font-semibold text-gray-800">Aplicaciones</h2>
            <button
              onClick={() => setIsAddingApp(true)}
              className="bg-indigo-600 hover:bg-indigo-700 text-white px-4 py-2 rounded-lg transition"
            >
              Añadir Aplicación
            </button>
          </div>
          {isAddingApp && (
            <div className="mb-6 p-4 bg-indigo-50 rounded-lg border border-indigo-100">
              <div className="flex flex-col space-y-3">
                <input
                  type="text"
                  placeholder="Nombre de la aplicación"
                  value={newAppName}
                  onChange={(e) => setNewAppName(e.target.value)}
                  className="px-4 py-2 rounded border border-gray-300 focus:outline-none focus:ring-2 focus:ring-indigo-300"
                />
                <input
                  type="number"
                  placeholder="Límite en minutos"
                  value={newAppLimit}
                  onChange={(e) => setNewAppLimit(parseInt(e.target.value))}
                  className="px-4 py-2 rounded border border-gray-300 focus:outline-none focus:ring-2 focus:ring-indigo-300"
                />
                <div className="flex space-x-2 mt-2">
                  <button
                    onClick={handleAddApp}
                    className="flex-1 bg-green-500 hover:bg-green-600 text-white py-2 rounded"
                  >
                    Guardar
                  </button>
                  <button
                    onClick={() => setIsAddingApp(false)}
                    className="flex-1 bg-gray-300 hover:bg-gray-400 text-gray-800 py-2 rounded"
                  >
                    Cancelar
                  </button>
                </div>
              </div>
            </div>
          )}
          <ul className="space-y-4">
            {filteredApps.length === 0 ? (
              <p className="text-gray-500 text-center py-4">No hay aplicaciones coincidentes.</p>
            ) : (
              filteredApps.map((app) => {
                const percentage = Math.min((app.used / app.limit) * 100, 100);
                let progressColor = "bg-green-500";
                if (percentage >= 80 && percentage < 100) progressColor = "bg-yellow-500";
                else if (percentage >= 100) progressColor = "bg-red-500";
                return (
                  <li key={app.id} className="border border-gray-200 rounded-lg p-4 relative">
                    <div className="flex justify-between items-start">
                      <h3 className="font-medium text-lg">{app.name}</h3>
                      <div className="flex space-x-2">
                        <button
                          onClick={() => resetUsage(app.id)}
                          className="text-xs text-indigo-600 hover:text-indigo-800"
                        >
                          Reiniciar
                        </button>
                        <button
                          onClick={() => setIsConfirmingDelete(app.id)}
                          className="text-xs text-red-500 hover:text-red-700"
                        >
                          Eliminar
                        </button>
                      </div>
                    </div>
                    <div className="mt-2">
                      <div className="flex justify-between text-sm text-gray-600 mb-1">
                        <span>Usado: {formatTime(app.used)}</span>
                        <span>Límite: {formatTime(app.limit)}</span>
                      </div>
                      <div className="w-full bg-gray-200 rounded-full h-2.5">
                        <div
                          className={`${progressColor} h-2.5 rounded-full transition-all duration-500 ease-out`}
                          style={{ width: `${percentage}%` }}
                        ></div>
                      </div>
                    </div>
                    {percentage >= 100 && (
                      <div className="mt-2 text-xs text-red-600 flex items-center">
                        <svg
                          xmlns="http://www.w3.org/2000/svg"
                          className="h-4 w-4 mr-1"
                          fill="none"
                          viewBox="0 0 24 24"
                          stroke="currentColor"
                        >
                          <path
                            strokeLinecap="round"
                            strokeLinejoin="round"
                            strokeWidth={2}
                            d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
                          />
                        </svg>
                        Has excedido el límite diario
                      </div>
                    )}
                  </li>
                );
              })
            )}
          </ul>
        </div>

        {/* Modal para cambiar contraseña */}
        {showChangePassword && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
            <div className="bg-white rounded-lg shadow-lg p-6 w-80">
              <h3 className="text-lg font-semibold mb-2">Cambiar Contraseña</h3>
              <input
                type="password"
                value={newPassword}
                onChange={(e) => setNewPassword(e.target.value)}
                placeholder="Nueva contraseña"
                className="w-full px-3 py-2 border border-gray-300 rounded mb-4 focus:outline-none focus:ring-2 focus:ring-indigo-300"
              />
              <div className="flex justify-end space-x-2">
                <button
                  onClick={() => setShowChangePassword(false)}
                  className="px-4 py-2 bg-gray-300 text-gray-700 rounded hover:bg-gray-400"
                >
                  Cancelar
                </button>
                <button
                  onClick={changePassword}
                  className="px-4 py-2 bg-indigo-600 text-white rounded hover:bg-indigo-700"
                >
                  Guardar
                </button>
              </div>
            </div>
          </div>
        )}

        {/* Modal para confirmar eliminación */}
        {isConfirmingDelete && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
            <div className="bg-white rounded-lg shadow-lg p-6 w-80">
              <h3 className="text-lg font-semibold mb-2">Confirmar Eliminación</h3>
              <p className="text-sm text-gray-600 mb-4">¿Estás seguro de que deseas eliminar esta aplicación?</p>
              <input
                type="password"
                value={deletePassword}
                onChange={(e) => setDeletePassword(e.target.value)}
                placeholder="Confirma tu contraseña"
                className="w-full px-3 py-2 border border-gray-300 rounded mb-4 focus:outline-none focus:ring-2 focus:ring-indigo-300"
              />
              <div className="flex justify-end space-x-2">
                <button
                  onClick={() => setIsConfirmingDelete(null)}
                  className="px-4 py-2 bg-gray-300 text-gray-700 rounded hover:bg-gray-400"
                >
                  Cancelar
                </button>
                <button
                  onClick={deleteApp}
                  className="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600"
                >
                  Eliminar
                </button>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
