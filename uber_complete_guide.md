# Guía Completa: Sistema Tipo Uber - Arquitectura y Desarrollo

## 1. ARQUITECTURA DEL BACKEND COMPLETA

### Microservicios Recomendados

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  User Service   │  │ Driver Service  │  │  Trip Service   │
│                 │  │                 │  │                 │
│ - Auth          │  │ - Registration  │  │ - Matching      │
│ - Profile       │  │ - Status        │  │ - Tracking      │
│ - Preferences   │  │ - Location      │  │ - Pricing       │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Payment Service │  │Location Service │  │Notification Svc │
│                 │  │                 │  │                 │
│ - Stripe        │  │ - GPS Tracking  │  │ - Push Notif    │
│ - Billing       │  │ - Geofencing    │  │ - SMS           │
│ - Invoices      │  │ - Route Calc    │  │ - Email         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Stack Tecnológico Backend

**API Gateway:** Kong, AWS API Gateway, o Nginx
**Servicios:** Node.js + Express o Python + FastAPI
**Base de Datos:** 
- PostgreSQL (datos relacionales)
- MongoDB (logs, ubicaciones)
- Redis (caché, sesiones)
**Message Queue:** RabbitMQ o Apache Kafka
**Real-time:** Socket.io o WebSockets nativos

### Modelos de Base de Datos

```sql
-- Usuarios
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE,
    name VARCHAR(100) NOT NULL,
    avatar_url TEXT,
    rating DECIMAL(3,2) DEFAULT 5.00,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Conductores
CREATE TABLE drivers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    license_number VARCHAR(50) UNIQUE NOT NULL,
    vehicle_type VARCHAR(20) NOT NULL,
    vehicle_plate VARCHAR(20) NOT NULL,
    is_active BOOLEAN DEFAULT false,
    is_available BOOLEAN DEFAULT false,
    current_lat DECIMAL(10,8),
    current_lng DECIMAL(11,8),
    last_location_update TIMESTAMP,
    rating DECIMAL(3,2) DEFAULT 5.00,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Viajes
CREATE TABLE trips (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    driver_id UUID REFERENCES drivers(id),
    status VARCHAR(20) DEFAULT 'requested',
    pickup_lat DECIMAL(10,8) NOT NULL,
    pickup_lng DECIMAL(11,8) NOT NULL,
    pickup_address TEXT NOT NULL,
    destination_lat DECIMAL(10,8) NOT NULL,
    destination_lng DECIMAL(11,8) NOT NULL,
    destination_address TEXT NOT NULL,
    estimated_fare DECIMAL(10,2),
    final_fare DECIMAL(10,2),
    distance_km DECIMAL(8,2),
    duration_minutes INTEGER,
    requested_at TIMESTAMP DEFAULT NOW(),
    accepted_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    cancelled_at TIMESTAMP
);
```

### APIs Principales

```javascript
// Estructura de endpoints
/api/v1/auth/
  POST /login
  POST /register
  POST /verify-phone
  POST /refresh-token

/api/v1/users/
  GET /profile
  PUT /profile
  GET /trip-history

/api/v1/drivers/
  POST /register
  PUT /status
  PUT /location
  GET /earnings

/api/v1/trips/
  POST /request
  PUT /:id/accept
  PUT /:id/start
  PUT /:id/complete
  DELETE /:id/cancel

/api/v1/locations/
  GET /nearby-drivers
  POST /track-trip
```

## 2. SISTEMA DE GEOLOCALIZACIÓN EN TIEMPO REAL

### Algoritmo de Matching Conductor-Pasajero

```python
import math
from typing import List, Tuple

def haversine_distance(lat1: float, lon1: float, lat2: float, lon2: float) -> float:
    """Calcula distancia entre dos puntos GPS en km"""
    R = 6371  # Radio de la Tierra en km
    
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    
    a = (math.sin(dlat/2) * math.sin(dlat/2) + 
         math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * 
         math.sin(dlon/2) * math.sin(dlon/2))
    
    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))
    distance = R * c
    
    return distance

def find_nearby_drivers(user_lat: float, user_lng: float, 
                       drivers: List[dict], max_distance: float = 5.0) -> List[dict]:
    """Encuentra conductores cercanos ordenados por distancia"""
    nearby_drivers = []
    
    for driver in drivers:
        if not driver['is_available']:
            continue
            
        distance = haversine_distance(
            user_lat, user_lng,
            driver['current_lat'], driver['current_lng']
        )
        
        if distance <= max_distance:
            driver['distance'] = round(distance, 2)
            nearby_drivers.append(driver)
    
    # Ordenar por distancia y rating
    nearby_drivers.sort(key=lambda x: (x['distance'], -x['rating']))
    
    return nearby_drivers[:10]  # Máximo 10 conductores

# Algoritmo de asignación inteligente
def smart_driver_assignment(user_location: dict, drivers: List[dict]) -> dict:
    """Asigna el mejor conductor considerando múltiples factores"""
    scored_drivers = []
    
    for driver in drivers:
        distance = haversine_distance(
            user_location['lat'], user_location['lng'],
            driver['current_lat'], driver['current_lng']
        )
        
        # Score basado en distancia, rating y tiempo de espera
        distance_score = max(0, 10 - distance)  # Máximo 10 puntos
        rating_score = driver['rating'] * 2     # Máximo 10 puntos
        availability_score = 5  # Puntos por estar disponible
        
        total_score = distance_score + rating_score + availability_score
        
        scored_drivers.append({
            'driver': driver,
            'score': total_score,
            'distance': distance
        })
    
    # Retorna el conductor con mejor score
    best_match = max(scored_drivers, key=lambda x: x['score'])
    return best_match['driver']
```

### Sistema de Tracking GPS

```javascript
// Cliente (App móvil)
class LocationTracker {
    constructor(tripId, socket) {
        this.tripId = tripId;
        this.socket = socket;
        this.watchId = null;
        this.lastUpdate = null;
    }

    startTracking() {
        const options = {
            enableHighAccuracy: true,
            timeout: 5000,
            maximumAge: 0
        };

        this.watchId = navigator.geolocation.watchPosition(
            (position) => this.handleLocationUpdate(position),
            (error) => this.handleLocationError(error),
            options
        );
    }

    handleLocationUpdate(position) {
        const now = Date.now();
        
        // Throttle actualizaciones (máximo cada 3 segundos)
        if (this.lastUpdate && now - this.lastUpdate < 3000) {
            return;
        }

        const locationData = {
            tripId: this.tripId,
            latitude: position.coords.latitude,
            longitude: position.coords.longitude,
            accuracy: position.coords.accuracy,
            speed: position.coords.speed,
            heading: position.coords.heading,
            timestamp: now
        };

        this.socket.emit('location_update', locationData);
        this.lastUpdate = now;
    }

    stopTracking() {
        if (this.watchId) {
            navigator.geolocation.clearWatch(this.watchId);
            this.watchId = null;
        }
    }
}

// Servidor (Backend)
io.on('connection', (socket) => {
    socket.on('location_update', async (data) => {
        try {
            // Validar datos
            if (!data.tripId || !data.latitude || !data.longitude) {
                return;
            }

            // Guardar en Redis para acceso rápido
            await redis.setex(
                `trip_location:${data.tripId}`,
                300, // 5 minutos TTL
                JSON.stringify(data)
            );

            // Enviar actualización a todos los participantes del viaje
            socket.to(`trip_${data.tripId}`).emit('location_updated', data);

            // Calcular ETA si es necesario
            if (data.speed && data.speed > 0) {
                const eta = await calculateETA(data);
                socket.to(`trip_${data.tripId}`).emit('eta_updated', eta);
            }

        } catch (error) {
            console.error('Error updating location:', error);
        }
    });
});
```

## 3. APP DE USUARIO (DESARROLLO DETALLADO)

### Estructura de Pantallas React Native

```javascript
// Navegación principal
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

const Stack = createStackNavigator();

export default function App() {
    return (
        <NavigationContainer>
            <Stack.Navigator initialRouteName="Login">
                <Stack.Screen name="Login" component={LoginScreen} />
                <Stack.Screen name="Home" component={HomeScreen} />
                <Stack.Screen name="BookRide" component={BookRideScreen} />
                <Stack.Screen name="TripTracking" component={TripTrackingScreen} />
                <Stack.Screen name="Payment" component={PaymentScreen} />
                <Stack.Screen name="History" component={HistoryScreen} />
            </Stack.Navigator>
        </NavigationContainer>
    );
}

// Pantalla principal de solicitud de viaje
import React, { useState, useEffect } from 'react';
import { View, Text, StyleSheet, TouchableOpacity } from 'react-native';
import MapView, { Marker, PROVIDER_GOOGLE } from 'react-native-maps';

const HomeScreen = ({ navigation }) => {
    const [currentLocation, setCurrentLocation] = useState(null);
    const [destination, setDestination] = useState(null);
    const [nearbyDrivers, setNearbyDrivers] = useState([]);

    useEffect(() => {
        getCurrentLocation();
        fetchNearbyDrivers();
    }, []);

    const getCurrentLocation = () => {
        navigator.geolocation.getCurrentPosition(
            (position) => {
                setCurrentLocation({
                    latitude: position.coords.latitude,
                    longitude: position.coords.longitude,
                    latitudeDelta: 0.01,
                    longitudeDelta: 0.01,
                });
            },
            (error) => console.error(error),
            { enableHighAccuracy: true, timeout: 20000, maximumAge: 1000 }
        );
    };

    const fetchNearbyDrivers = async () => {
        try {
            const response = await fetch('/api/v1/locations/nearby-drivers', {
                method: 'GET',
                headers: {
                    'Authorization': `Bearer ${userToken}`,
                    'Content-Type': 'application/json',
                },
            });
            const drivers = await response.json();
            setNearbyDrivers(drivers);
        } catch (error) {
            console.error('Error fetching drivers:', error);
        }
    };

    const requestRide = async () => {
        if (!destination) {
            alert('Por favor selecciona un destino');
            return;
        }

        try {
            const response = await fetch('/api/v1/trips/request', {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${userToken}`,
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    pickup_lat: currentLocation.latitude,
                    pickup_lng: currentLocation.longitude,
                    destination_lat: destination.latitude,
                    destination_lng: destination.longitude,
                }),
            });

            const trip = await response.json();
            navigation.navigate('TripTracking', { tripId: trip.id });
        } catch (error) {
            console.error('Error requesting ride:', error);
        }
    };

    return (
        <View style={styles.container}>
            <MapView
                provider={PROVIDER_GOOGLE}
                style={styles.map}
                region={currentLocation}
                onPress={(event) => setDestination(event.nativeEvent.coordinate)}
            >
                {currentLocation && (
                    <Marker
                        coordinate={currentLocation}
                        title="Mi ubicación"
                        pinColor="blue"
                    />
                )}
                
                {destination && (
                    <Marker
                        coordinate={destination}
                        title="Destino"
                        pinColor="red"
                    />
                )}

                {nearbyDrivers.map((driver) => (
                    <Marker
                        key={driver.id}
                        coordinate={{
                            latitude: driver.current_lat,
                            longitude: driver.current_lng,
                        }}
                        title={`Conductor - ${driver.rating}⭐`}
                        pinColor="green"
                    />
                ))}
            </MapView>

            <View style={styles.bottomSheet}>
                <TouchableOpacity
                    style={styles.requestButton}
                    onPress={requestRide}
                >
                    <Text style={styles.buttonText}>Solicitar Viaje</Text>
                </TouchableOpacity>
            </View>
        </View>
    );
};

const styles = StyleSheet.create({
    container: {
        flex: 1,
    },
    map: {
        flex: 1,
    },
    bottomSheet: {
        position: 'absolute',
        bottom: 0,
        left: 0,
        right: 0,
        backgroundColor: 'white',
        padding: 20,
        borderTopLeftRadius: 20,
        borderTopRightRadius: 20,
    },
    requestButton: {
        backgroundColor: '#007AFF',
        padding: 15,
        borderRadius: 10,
        alignItems: 'center',
    },
    buttonText: {
        color: 'white',
        fontSize: 18,
        fontWeight: 'bold',
    },
});
```

### Integración de Pagos

```javascript
// Componente de pago con Stripe
import React, { useState } from 'react';
import { CardField, useStripe } from '@stripe/stripe-react-native';

const PaymentScreen = ({ route, navigation }) => {
    const { tripId, amount } = route.params;
    const { confirmPayment } = useStripe();
    const [loading, setLoading] = useState(false);

    const handlePayment = async () => {
        setLoading(true);
        
        try {
            // Crear Payment Intent en el backend
            const response = await fetch('/api/v1/payments/create-intent', {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${userToken}`,
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    tripId,
                    amount: amount * 100, // Convertir a centavos
                }),
            });

            const { clientSecret } = await response.json();

            // Confirmar pago
            const { error, paymentIntent } = await confirmPayment(clientSecret, {
                paymentMethodType: 'Card',
            });

            if (error) {
                alert(`Error en el pago: ${error.message}`);
            } else if (paymentIntent) {
                alert('Pago exitoso!');
                navigation.navigate('TripComplete', { tripId });
            }
        } catch (error) {
            alert('Error procesando el pago');
        } finally {
            setLoading(false);
        }
    };

    return (
        <View style={styles.container}>
            <Text style={styles.title}>Total a pagar: ${amount}</Text>
            
            <CardField
                postalCodeEnabled={false}
                placeholder={{
                    number: '4242 4242 4242 4242',
                }}
                cardStyle={{
                    backgroundColor: '#FFFFFF',
                    textColor: '#000000',
                }}
                style={styles.cardField}
            />

            <TouchableOpacity
                style={[styles.payButton, loading && styles.disabled]}
                onPress={handlePayment}
                disabled={loading}
            >
                <Text style={styles.payButtonText}>
                    {loading ? 'Procesando...' : 'Pagar'}
                </Text>
            </TouchableOpacity>
        </View>
    );
};
```

## 4. APP DE CONDUCTOR (DESARROLLO DETALLADO)

### Estados del Conductor

```javascript
// Hook para manejar estados del conductor
import { useState, useEffect } from 'react';
import { useSocket } from './useSocket';

const useDriverStatus = () => {
    const [status, setStatus] = useState('offline'); // offline, online, busy, in_trip
    const [currentTrip, setCurrentTrip] = useState(null);
    const [isAvailable, setIsAvailable] = useState(false);
    const socket = useSocket();

    useEffect(() => {
        // Escuchar solicitudes de viaje
        socket.on('trip_request', handleTripRequest);
        
        return () => {
            socket.off('trip_request', handleTripRequest);
        };
    }, []);

    const handleTripRequest = (tripData) => {
        if (status === 'online' && isAvailable) {
            // Mostrar notificación de solicitud
            showTripRequestNotification(tripData);
        }
    };

    const goOnline = async () => {
        try {
            await updateDriverStatus('online');
            setStatus('online');
            setIsAvailable(true);
            startLocationTracking();
        } catch (error) {
            console.error('Error going online:', error);
        }
    };

    const goOffline = async () => {
        try {
            await updateDriverStatus('offline');
            setStatus('offline');
            setIsAvailable(false);
            stopLocationTracking();
        } catch (error) {
            console.error('Error going offline:', error);
        }
    };

    const acceptTrip = async (tripId) => {
        try {
            const response = await fetch(`/api/v1/trips/${tripId}/accept`, {
                method: 'PUT',
                headers: {
                    'Authorization': `Bearer ${driverToken}`,
                    'Content-Type': 'application/json',
                },
            });

            const trip = await response.json();
            setCurrentTrip(trip);
            setStatus('in_trip');
            setIsAvailable(false);
        } catch (error) {
            console.error('Error accepting trip:', error);
        }
    };

    return {
        status,
        currentTrip,
        isAvailable,
        goOnline,
        goOffline,
        acceptTrip,
    };
};

// Pantalla principal del conductor
const DriverHomeScreen = () => {
    const { status, currentTrip, goOnline, goOffline, acceptTrip } = useDriverStatus();
    const [earnings, setEarnings] = useState({ today: 0, week: 0 });

    const TripRequestModal = ({ trip, onAccept, onDecline }) => (
        <Modal visible={!!trip} animationType="slide">
            <View style={styles.modalContainer}>
                <Text style={styles.modalTitle}>Nueva Solicitud de Viaje</Text>
                <Text>Desde: {trip?.pickup_address}</Text>
                <Text>Hasta: {trip?.destination_address}</Text>
                <Text>Distancia: {trip?.distance} km</Text>
                <Text>Tarifa estimada: ${trip?.estimated_fare}</Text>
                
                <View style={styles.modalButtons}>
                    <TouchableOpacity
                        style={[styles.button, styles.declineButton]}
                        onPress={onDecline}
                    >
                        <Text>Rechazar</Text>
                    </TouchableOpacity>
                    
                    <TouchableOpacity
                        style={[styles.button, styles.acceptButton]}
                        onPress={() => onAccept(trip.id)}
                    >
                        <Text>Aceptar</Text>
                    </TouchableOpacity>
                </View>
            </View>
        </Modal>
    );

    return (
        <View style={styles.container}>
            <View style={styles.statusBar}>
                <Text style={styles.statusText}>
                    Estado: {status === 'online' ? 'En línea' : 'Fuera de línea'}
                </Text>
                
                <TouchableOpacity
                    style={[
                        styles.statusButton,
                        status === 'online' ? styles.onlineButton : styles.offlineButton
                    ]}
                    onPress={status === 'online' ? goOffline : goOnline}
                >
                    <Text style={styles.statusButtonText}>
                        {status === 'online' ? 'Desconectar' : 'Conectar'}
                    </Text>
                </TouchableOpacity>
            </View>

            <View style={styles.earningsCard}>
                <Text style={styles.earningsTitle}>Ganancias</Text>
                <Text style={styles.earningsAmount}>Hoy: ${earnings.today}</Text>
                <Text style={styles.earningsAmount}>Semana: ${earnings.week}</Text>
            </View>

            {currentTrip && (
                <TripInProgressComponent trip={currentTrip} />
            )}
        </View>
    );
};
```

## 5. DASHBOARD ADMINISTRATIVO COMPLETO

### Panel de Control en Tiempo Real

```javascript
// Dashboard principal con React
import React, { useState, useEffect } from 'react';
import { Line, Bar, Doughnut } from 'react-chartjs-2';
import io from 'socket.io-client';

const AdminDashboard = () => {
    const [stats, setStats] = useState({
        activeDrivers: 0,
        activeTrips: 0,
        totalUsers: 0,
        revenue: 0
    });
    const [realtimeTrips, setRealtimeTrips] = useState([]);
    const [socket, setSocket] = useState(null);

    useEffect(() => {
        // Conectar a socket para datos en tiempo real
        const socketConnection = io(process.env.REACT_APP_SOCKET_URL, {
            auth: { token: adminToken }
        });

        socketConnection.on('stats_update', (newStats) => {
            setStats(newStats);
        });

        socketConnection.on('trip_update', (trip) => {
            setRealtimeTrips(prev => {
                const index = prev.findIndex(t => t.id === trip.id);
                if (index >= 0) {
                    const updated = [...prev];
                    updated[index] = trip;
                    return updated;
                } else {
                    return [trip, ...prev];
                }
            });
        });

        setSocket(socketConnection);

        return () => socketConnection.close();
    }, []);

    const StatsCard = ({ title, value, icon, color }) => (
        <div className={`stats-card ${color}`}>
            <div className="stats-icon">{icon}</div>
            <div className="stats-content">
                <h3>{title}</h3>
                <span className="stats-value">{value}</span>
            </div>
        </div>
    );

    const RealtimeTripsTable = () => (
        <div className="realtime-trips">
            <h3>Viajes en Tiempo Real</h3>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Conductor</th>
                        <th>Usuario</th>
                        <th>Estado</th>
                        <th>Distancia</th>
                        <th>Tarifa</th>
                        <th>Tiempo</th>
                    </tr>
                </thead>
                <tbody>
                    {realtimeTrips.map(trip => (
                        <tr key={trip.id}>
                            <td>{trip.id.substring(0, 8)}</td>
                            <td>{trip.driver_name}</td>
                            <td>{trip.user_name}</td>
                            <td>
                                <span className={`status-badge ${trip.status}`}>
                                    {trip.status}
                                </span>
                            </td>
                            <td>{trip.distance} km</td>
                            <td>${trip.fare}</td>
                            <td>{trip.duration}min</td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );

    return (
        <div className="admin-dashboard">
            <header className="dashboard-header">
                <h1>Panel de Control - UberClone</h1>
            </header>

            <div className="stats-grid">
                <StatsCard
                    title="Conductores Activos"
                    value={stats.activeDrivers}
                    icon="🚗"
                    color="blue"
                />
                <StatsCard
                    title="Viajes Activos"
                    value={stats.activeTrips}
                    icon="🚦"
                    color="green"
                />
                <StatsCard
                    title="Total Usuarios"
                    value={stats.totalUsers.toLocaleString()}
                    icon="👥"
                    color="purple"
                />
                <StatsCard
                    title="Ingresos Hoy"
                    value={`$${stats.revenue.toLocaleString()}`}
                    icon="💰"
                    color="orange"
                />
            </div>

            <div className="dashboard-content">
                <div className="charts-section">
                    <RevenueChart />
                    <TripsChart />
                </div>
                
                <RealtimeTripsTable />
            </div>
        </div>
    );
};

// Componente de gestión de conductores
const DriversManagement = () => {
    const [drivers, setDrivers] = useState([]);
    const [pendingDrivers, setPendingDrivers] = useState([]);

    const approveDriver = async (driverId) => {
        try {
            await fetch(`/api/v1/admin/drivers/${driverId}/approve`, {
                method: 'PUT',
                headers: {
                    'Authorization': `Bearer ${adminToken}`,
                    'Content-Type': 'application/json',
                },
            });
            
            // Actualizar listas
            fetchDrivers();
        } catch (error) {
            console.error('Error approving driver:', error);
        }
    };

    const suspendDriver = async (driverId) => {
        try {
            await fetch(`/api/v1/admin/drivers/${driverId}/suspend`, {
                method: 'PUT',
                headers: {
                    'Authorization': `Bearer ${adminToken}`,
                    'Content-Type': 'application/json',
                },
            });
            
            fetchDrivers();
        } catch (error) {
            console.error('Error suspending driver:', error);
        }
    };

    return (
        <div className="drivers-management">
            <div className="pending-approvals">
                <h3>Conductores Pendientes de Aprobación</h3>
                {pendingDrivers.map(driver => (
                    <div key={driver.id} className="driver-card pending">
                        <div className="driver-info">
                            <h4>{driver.name}</h4>
                            <p>Licencia: {driver.license_number}</p>
                            <p>Vehículo: {driver.vehicle_type} - {driver.vehicle_plate}</p>
                        </div>
                        <div className="driver-actions">
                            <button
                                className="approve-btn"
                                onClick={() => approveDriver(driver.id)}
                            >
                                Aprobar
                            </button>
                            <button
                                className="reject-btn"
                                onClick={() => rejectDriver(driver.id)}
                            >
                                Rechazar
                            </button>
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
};
```

## 6. IMPLEMENTACIÓN TÉCNICA PASO A PASO

### Stack Tecnológico Específico

```json
{
  "backend": {
    "runtime": "Node.js 18+",
    "framework": "Express.js",
    "database": "PostgreSQL 14+",
    "cache": "Redis 7+",
    "realtime": "Socket.io",
    "queue": "Bull (Redis-based)",
    "auth": "JWT + bcrypt"
  },
  "mobile": {
    "framework": "React Native 0.72+",
    "maps": "react-native-maps",
    "payments": "@stripe/stripe-react-native",
    "navigation": "@react-navigation/native",
    "state": "Redux Toolkit",
    "realtime": "socket.io-client"
  },
  "web_dashboard": {
    "framework": "React 18+",
    "ui": "Material-UI / Tailwind CSS",
    "charts": "Chart.js / Recharts",
    "state": "Redux Toolkit"
  },
  "infrastructure": {
    "hosting": "AWS / Google Cloud",
    "cdn": "CloudFront / CloudFlare",
    "monitoring": "Sentry + DataDog",
    "ci_cd": "GitHub Actions"
  }
}
```

### Configuración del Proyecto

```bash
# 1. Crear estructura del proyecto
mkdir uber-clone
cd uber-clone

# Estructura de carpetas
mkdir -p backend/{src/{controllers,models,routes,middleware,services,utils},tests}
mkdir -p mobile-app/{src/{screens,components,services,utils,navigation},assets}
mkdir -p web-dashboard/{src/{components,pages,services,utils},public}
mkdir -p shared/{types,constants}

# 2. Inicializar Backend
cd backend
npm init -y
npm install express cors helmet morgan compression
npm install jsonwebtoken bcryptjs
npm install pg redis socket.io
npm install dotenv nodemon --save-dev

# 3. Configurar React Native
cd ../mobile-app
npx react-native init UberCloneApp
cd UberCloneApp
npm install @react-navigation/native @react-navigation/stack
npm install react-native-maps @stripe/stripe-react-native
npm install socket.io-client @reduxjs/toolkit react-redux

# 4. Configurar Dashboard Web
cd ../../web-dashboard
npx create-react-app uber-admin-dashboard
cd uber-admin-dashboard
npm install @mui/material @mui/icons-material chart.js react-chartjs-2
npm install socket.io-client @reduxjs/toolkit react-redux
```

### Configuración del Backend

```javascript
// backend/src/app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const compression = require('compression');
const { createServer } = require('http');
const { Server } = require('socket.io');
require('dotenv').config();

const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const driverRoutes = require('./routes/drivers');
const tripRoutes = require('./routes/trips');
const adminRoutes = require('./routes/admin');

const app = express();
const server = createServer(app);
const io = new Server(server, {
    cors: {
        origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
        methods: ['GET', 'POST']
    }
});

// Middleware
app.use(helmet());
app.use(cors());
app.use(compression());
app.use(morgan('combined'));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/users', userRoutes);
app.use('/api/v1/drivers', driverRoutes);
app.use('/api/v1/trips', tripRoutes);
app.use('/api/v1/admin', adminRoutes);

// Socket.io para tiempo real
require('./services/socketService')(io);

// Error handling
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ 
        message: 'Error interno del servidor',
        error: process.env.NODE_ENV === 'development' ? err.message : undefined
    });
});

const PORT = process.env.PORT || 3001;
server.listen(PORT, () => {
    console.log(`Servidor corriendo en puerto ${PORT}`);
});

module.exports = { app, io };
```

```javascript
// backend/src/config/database.js
const { Pool } = require('pg');
const Redis = require('redis');

// PostgreSQL connection
const pool = new Pool({
    user: process.env.DB_USER || 'postgres',
    host: process.env.DB_HOST || 'localhost',
    database: process.env.DB_NAME || 'uber_clone',
    password: process.env.DB_PASSWORD || 'password',
    port: process.env.DB_PORT || 5432,
    max: 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
});

// Redis connection
const redis = Redis.createClient({
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379,
    password: process.env.REDIS_PASSWORD
});

redis.on('error', (err) => {
    console.error('Redis error:', err);
});

redis.connect();

module.exports = { pool, redis };
```

```javascript
// backend/src/services/socketService.js
const jwt = require('jsonwebtoken');
const { redis } = require('../config/database');

module.exports = (io) => {
    // Middleware de autenticación para sockets
    io.use((socket, next) => {
        const token = socket.handshake.auth.token;
        
        if (!token) {
            return next(new Error('Authentication error'));
        }

        try {
            const decoded = jwt.verify(token, process.env.JWT_SECRET);
            socket.userId = decoded.id;
            socket.userType = decoded.type; // 'user', 'driver', 'admin'
            next();
        } catch (err) {
            next(new Error('Authentication error'));
        }
    });

    io.on('connection', (socket) => {
        console.log(`Usuario conectado: ${socket.userId} (${socket.userType})`);
        
        // Unir a room según tipo de usuario
        socket.join(socket.userType);
        
        if (socket.userType === 'driver') {
            socket.join('drivers');
            handleDriverConnection(socket, io);
        } else if (socket.userType === 'user') {
            handleUserConnection(socket, io);
        } else if (socket.userType === 'admin') {
            socket.join('admins');
            handleAdminConnection(socket, io);
        }

        socket.on('disconnect', () => {
            console.log(`Usuario desconectado: ${socket.userId}`);
            if (socket.userType === 'driver') {
                handleDriverDisconnection(socket);
            }
        });
    });
};

// Manejar conexión de conductores
const handleDriverConnection = (socket, io) => {
    // Actualizar ubicación del conductor
    socket.on('update_location', async (locationData) => {
        try {
            const driverLocation = {
                driverId: socket.userId,
                latitude: locationData.latitude,
                longitude: locationData.longitude,
                timestamp: Date.now()
            };

            // Guardar en Redis
            await redis.setex(
                `driver_location:${socket.userId}`,
                300, // 5 minutos TTL
                JSON.stringify(driverLocation)
            );

            // Notificar a administradores
            io.to('admins').emit('driver_location_updated', driverLocation);

        } catch (error) {
            console.error('Error updating driver location:', error);
        }
    });

    // Manejar aceptación de viaje
    socket.on('accept_trip', async (tripId) => {
        try {
            // Lógica para aceptar viaje
            const trip = await acceptTripService(tripId, socket.userId);
            
            // Notificar al usuario
            io.to(`trip_${tripId}`).emit('trip_accepted', trip);
            
            // Notificar a administradores
            io.to('admins').emit('trip_update', trip);

        } catch (error) {
            socket.emit('error', { message: 'Error aceptando viaje' });
        }
    });
};

// Manejar conexión de usuarios
const handleUserConnection = (socket, io) => {
    socket.on('request_trip', async (tripData) => {
        try {
            // Crear viaje en base de datos
            const trip = await createTripService(tripData, socket.userId);
            
            // Unir al room del viaje
            socket.join(`trip_${trip.id}`);
            
            // Buscar conductores cercanos
            const nearbyDrivers = await findNearbyDriversService(
                tripData.pickup_lat,
                tripData.pickup_lng
            );

            // Enviar solicitud a conductores cercanos
            nearbyDrivers.forEach(driver => {
                io.to(driver.socketId).emit('trip_request', {
                    tripId: trip.id,
                    pickup_address: tripData.pickup_address,
                    destination_address: tripData.destination_address,
                    estimated_fare: trip.estimated_fare,
                    distance: trip.distance_km
                });
            });

            socket.emit('trip_requested', { tripId: trip.id });

        } catch (error) {
            socket.emit('error', { message: 'Error solicitando viaje' });
        }
    });
};

// Manejar conexión de administradores
const handleAdminConnection = (socket, io) => {
    socket.on('get_realtime_stats', async () => {
        try {
            const stats = await getRealtimeStatsService();
            socket.emit('realtime_stats', stats);
        } catch (error) {
            socket.emit('error', { message: 'Error obteniendo estadísticas' });
        }
    });
};
```

### Servicios Backend

```javascript
// backend/src/services/tripService.js
const { pool } = require('../config/database');
const { calculateDistance, calculateFare } = require('../utils/calculations');

class TripService {
    async createTrip(tripData, userId) {
        const client = await pool.connect();
        
        try {
            await client.query('BEGIN');

            // Calcular distancia y tarifa
            const distance = calculateDistance(
                tripData.pickup_lat,
                tripData.pickup_lng,
                tripData.destination_lat,
                tripData.destination_lng
            );

            const estimatedFare = calculateFare(distance);

            // Insertar viaje
            const tripQuery = `
                INSERT INTO trips (
                    user_id, pickup_lat, pickup_lng, pickup_address,
                    destination_lat, destination_lng, destination_address,
                    distance_km, estimated_fare, status
                ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
                RETURNING *
            `;

            const result = await client.query(tripQuery, [
                userId,
                tripData.pickup_lat,
                tripData.pickup_lng,
                tripData.pickup_address,
                tripData.destination_lat,
                tripData.destination_lng,
                tripData.destination_address,
                distance,
                estimatedFare,
                'requested'
            ]);

            await client.query('COMMIT');
            return result.rows[0];

        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }

    async acceptTrip(tripId, driverId) {
        const client = await pool.connect();
        
        try {
            await client.query('BEGIN');

            // Verificar que el viaje esté disponible
            const tripCheck = await client.query(
                'SELECT * FROM trips WHERE id = $1 AND status = $2',
                [tripId, 'requested']
            );

            if (tripCheck.rows.length === 0) {
                throw new Error('Viaje no disponible');
            }

            // Actualizar viaje con conductor
            const updateQuery = `
                UPDATE trips 
                SET driver_id = $1, status = $2, accepted_at = NOW()
                WHERE id = $3
                RETURNING *
            `;

            const result = await client.query(updateQuery, [
                driverId,
                'accepted',
                tripId
            ]);

            await client.query('COMMIT');
            return result.rows[0];

        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }

    async updateTripStatus(tripId, status, driverId = null) {
        const validStatuses = ['accepted', 'started', 'completed', 'cancelled'];
        
        if (!validStatuses.includes(status)) {
            throw new Error('Estado inválido');
        }

        const timestampField = `${status}_at`;
        const query = `
            UPDATE trips 
            SET status = $1, ${timestampField} = NOW()
            WHERE id = $2 ${driverId ? 'AND driver_id = $3' : ''}
            RETURNING *
        `;

        const params = driverId ? [status, tripId, driverId] : [status, tripId];
        const result = await pool.query(query, params);

        return result.rows[0];
    }

    async getTripsByUser(userId, limit = 20, offset = 0) {
        const query = `
            SELECT t.*, d.name as driver_name, d.rating as driver_rating,
                   u.name as user_name
            FROM trips t
            LEFT JOIN drivers dr ON t.driver_id = dr.id
            LEFT JOIN users d ON dr.user_id = d.id
            LEFT JOIN users u ON t.user_id = u.id
            WHERE t.user_id = $1
            ORDER BY t.created_at DESC
            LIMIT $2 OFFSET $3
        `;

        const result = await pool.query(query, [userId, limit, offset]);
        return result.rows;
    }

    async getTripsByDriver(driverId, limit = 20, offset = 0) {
        const query = `
            SELECT t.*, u.name as user_name, u.rating as user_rating
            FROM trips t
            LEFT JOIN users u ON t.user_id = u.id
            WHERE t.driver_id = (
                SELECT id FROM drivers WHERE user_id = $1
            )
            ORDER BY t.created_at DESC
            LIMIT $2 OFFSET $3
        `;

        const result = await pool.query(query, [driverId, limit, offset]);
        return result.rows;
    }
}

module.exports = new TripService();
```

```javascript
// backend/src/services/locationService.js
const { redis } = require('../config/database');
const { calculateDistance } = require('../utils/calculations');

class LocationService {
    async updateDriverLocation(driverId, latitude, longitude) {
        const locationData = {
            driverId,
            latitude,
            longitude,
            timestamp: Date.now()
        };

        // Guardar en Redis con TTL de 5 minutos
        await redis.setex(
            `driver_location:${driverId}`,
            300,
            JSON.stringify(locationData)
        );

        // Agregar a índice geoespacial (usando sorted set)
        const geoKey = 'drivers_locations';
        await redis.geoadd(geoKey, longitude, latitude, driverId);

        return locationData;
    }

    async findNearbyDrivers(latitude, longitude, radiusKm = 5, limit = 10) {
        try {
            // Buscar conductores en radio usando GEORADIUS
            const nearbyDriverIds = await redis.georadius(
                'drivers_locations',
                longitude,
                latitude,
                radiusKm,
                'km',
                'WITHDIST',
                'ASC',
                'COUNT',
                limit
            );

            const nearbyDrivers = [];

            for (const [driverId, distance] of nearbyDriverIds) {
                // Obtener datos del conductor desde Redis
                const locationData = await redis.get(`driver_location:${driverId}`);
                
                if (locationData) {
                    const driverLocation = JSON.parse(locationData);
                    
                    // Verificar que la ubicación no sea muy antigua (5 minutos)
                    const isLocationFresh = Date.now() - driverLocation.timestamp < 300000;
                    
                    if (isLocationFresh) {
                        nearbyDrivers.push({
                            driverId,
                            distance: parseFloat(distance),
                            ...driverLocation
                        });
                    }
                }
            }

            return nearbyDrivers;

        } catch (error) {
            console.error('Error finding nearby drivers:', error);
            return [];
        }
    }

    async getDriverLocation(driverId) {
        const locationData = await redis.get(`driver_location:${driverId}`);
        return locationData ? JSON.parse(locationData) : null;
    }

    async removeDriverLocation(driverId) {
        await redis.del(`driver_location:${driverId}`);
        await redis.zrem('drivers_locations', driverId);
    }

    // Calcular ETA basado en tráfico y distancia
    async calculateETA(fromLat, fromLng, toLat, toLng) {
        const distance = calculateDistance(fromLat, fromLng, toLat, toLng);
        
        // Velocidad promedio en ciudad (considerando tráfico)
        const avgSpeedKmh = 25;
        const etaMinutes = Math.ceil((distance / avgSpeedKmh) * 60);
        
        return {
            distance_km: distance,
            eta_minutes: etaMinutes,
            estimated_arrival: new Date(Date.now() + etaMinutes * 60000)
        };
    }
}

module.exports = new LocationService();
```

### Utilidades y Helpers

```javascript
// backend/src/utils/calculations.js
const haversineDistance = (lat1, lon1, lat2, lon2) => {
    const R = 6371; // Radio de la Tierra en km
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;
    
    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
              Math.sin(dLon/2) * Math.sin(dLon/2);
    
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    return R * c;
};

const calculateFare = (distanceKm, baseRate = 2.5, perKmRate = 1.8) => {
    const baseFare = baseRate;
    const distanceFare = distanceKm * perKmRate;
    const totalFare = baseFare + distanceFare;
    
    // Tarifa mínima
    const minimumFare = 5.0;
    
    return Math.max(totalFare, minimumFare);
};

const calculateSurge = (demandRatio) => {
    // Surge pricing basado en demanda
    if (demandRatio < 1.2) return 1.0;  // Sin surge
    if (demandRatio < 1.5) return 1.3;  // Surge bajo
    if (demandRatio < 2.0) return 1.6;  // Surge medio
    return 2.0;  // Surge alto
};

module.exports = {
    calculateDistance: haversineDistance,
    calculateFare,
    calculateSurge
};
```

### Configuración Docker

```dockerfile
# backend/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copiar package files
COPY package*.json ./
RUN npm ci --only=production

# Copiar código fuente
COPY src/ ./src/

# Crear usuario no-root
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

EXPOSE 3001

CMD ["node", "src/app.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Base de datos PostgreSQL
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: uber_clone
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backend/migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"

  # Redis para caché y sesiones
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

  # Backend API
  backend:
    build: ./backend
    environment:
      NODE_ENV: development
      DB_HOST: postgres
      DB_USER: postgres
      DB_PASSWORD: password
      DB_NAME: uber_clone
      REDIS_HOST: redis
      JWT_SECRET: your-jwt-secret
      STRIPE_SECRET_KEY: your-stripe-secret
    ports:
      - "3001:3001"
    depends_on:
      - postgres
      - redis
    volumes:
      - ./backend:/app
      - /app/node_modules

  # Dashboard Web
  web-dashboard:
    build: ./web-dashboard
    ports:
      - "3000:3000"
    environment:
      REACT_APP_API_URL: http://localhost:3001
      REACT_APP_SOCKET_URL: http://localhost:3001
    volumes:
      - ./web-dashboard:/app
      - /app/node_modules

volumes:
  postgres_data:
  redis_data:
```

### Deploy y Hosting

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: |
        cd backend && npm ci
        cd ../web-dashboard && npm ci
    
    - name: Run tests
      run: |
        cd backend && npm test
        cd ../web-dashboard && npm test
    
    - name: Build applications
      run: |
        cd web-dashboard && npm run build
    
    - name: Deploy to AWS
      run: |
        # Configurar AWS CLI
        aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws configure set default.region us-east-1
        
        # Deploy backend to ECS
        aws ecs update-service --cluster uber-clone --service backend --force-new-deployment
        
        # Deploy dashboard to S3/CloudFront
        aws s3 sync web-dashboard/build s3://uber-clone-dashboard --delete
        aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_ID }} --paths "/*"
```

### Configuración de Producción

```javascript
// backend/src/config/production.js
module.exports = {
    database: {
        host: process.env.RDS_HOSTNAME,
        port: process.env.RDS_PORT,
        database: process.env.RDS_DB_NAME,
        username: process.env.RDS_USERNAME,
        password: process.env.RDS_PASSWORD,
        ssl: true,
        pool: {
            min: 2,
            max: 20,
            acquire: 30000,
            idle: 10000
        }
    },
    redis: {
        host: process.env.ELASTICACHE_ENDPOINT,
        port: 6379,
        family: 4,
        keepAlive: true,
        maxRetriesPerRequest: 3
    },
    security: {
        jwtSecret: process.env.JWT_SECRET,
        encryptionKey: process.env.ENCRYPTION_KEY,
        corsOrigins: process.env.CORS_ORIGINS.split(',')
    },
    services: {
        stripe: {
            secretKey: process.env.STRIPE_SECRET_KEY,
            webhookSecret: process.env.STRIPE_WEBHOOK_SECRET
        },
        maps: {
            googleApiKey: process.env.GOOGLE_MAPS_API_KEY
        },
        notifications: {
            firebaseServerKey: process.env.FIREBASE_SERVER_KEY,
            twilioAccountSid: process.env.TWILIO_ACCOUNT_SID,
            twilioAuthToken: process.env.TWILIO_AUTH_TOKEN
        }
    }
};
```

### Monitoreo y Logging

```javascript
// backend/src/middleware/monitoring.js
const winston = require('winston');
const CloudWatchTransport = require('winston-cloudwatch');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
        new winston.transports.File({ filename: 'logs/combined.log' }),
        new CloudWatchTransport({
            logGroupName: 'uber-clone-backend',
            logStreamName: `${process.env.NODE_ENV}-${Date.now()}`,
            awsRegion: 'us-east-1'
        })
    ]
});

const requestLogger = (req, res, next) => {
    const start = Date.now();
    
    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info('HTTP Request', {
            method: req.method,
            url: req.url,
            status: res.statusCode,
            duration,
            userAgent: req.get('User-Agent'),
            ip: req.ip,
            userId: req.user?.id
        });
    });
    
    next();
};

module.exports = { logger, requestLogger };
```

## Consideraciones Finales de Implementación

### Seguridad
- Implementar rate limiting para APIs
- Validación estricta de inputs
- Encriptación de datos sensibles
- Autenticación de dos factores
- Auditoría de accesos

### Performance
- Implementar CDN para assets estáticos
- Usar compression gzip/deflate
- Optimizar queries de base de datos
- Implementar caché en múltiples niveles
- Lazy loading en aplicaciones móviles

### Escalabilidad
- Arquitectura de microservicios
- Load balancers para distribuir tráfico
- Auto-scaling basado en métricas
- Base de datos con read replicas
- Queue system para procesamiento asíncrono

### Compliance y Legal
- Cumplir con GDPR/CCPA para privacidad
- Implementar políticas de retención de datos
- Auditorías de seguridad regulares
- Términos y condiciones claros
- Políticas de cancelación y reembolsos

Este sistema completo te proporciona una base sólida para construir una aplicación tipo Uber con todas las funcionalidades esenciales y la arquitectura necesaria para escalar.