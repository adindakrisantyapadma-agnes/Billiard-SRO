# Billiard-SRO
# Adinda_Krisantya_Padma # 5022221156

from coppeliasim_zmqremoteapi_client import RemoteAPIClient
import time
import math
import numpy as np

class BilliardGame:
    def __init__(self):
        print("Memulai Permainan Biliar...")
        
        # Koneksi ke CoppeliaSim
        self.client = RemoteAPIClient()
        self.sim = self.client.getObject('sim')
        
        # Setup simulasi
        self.setup_simulation()
        
        # Dapatkan handle objek
        self.get_object_handles()
        
        # Setup parameter meja
        self.setup_table_parameters()
        
        # Posisi awal bola putih
        self.white_ball_start_pos = [-0.5, 0, 0.03]
        
        # Status permainan
        self.game_active = True
        
    def setup_simulation(self):
        """Setup simulasi CoppeliaSim"""
        self.sim.stopSimulation()
        time.sleep(0.5)
        self.sim.setStepping(True)
        self.sim.startSimulation()
        time.sleep(0.5)
    
    def get_object_handles(self):
        """Dapatkan handle untuk semua objek yang diperlukan"""
        print("Mencari objek-objek dalam scene...")
        
        # Cari bola putih
        self.white_ball = None
        possible_names = ["/CueBall", "/WhiteBall", "/Sphere[0]", "/Sphere0", "/Sphere"]
        
        for name in possible_names:
            try:
                self.white_ball = self.sim.getObject(name)
                print(f"Found white ball: {name}")
                break
            except:
                continue
                
        if self.white_ball is None:
            # Jika tidak ditemukan, cari semua sphere dan ambil yang pertama sebagai white ball
            all_objects = self.sim.getObjects(self.sim.handle_all)
            for obj in all_objects:
                try:
                    obj_type = self.sim.getObjectType(obj)
                    if obj_type == self.sim.object_shape_type:
                        self.white_ball = obj
                        print(f"Using object as white ball: {self.sim.getObjectAlias(obj, 1)}")
                        break
                except:
                    continue
        
        # Cari bola lainnya
        self.balls = []
        try:
            all_objects = self.sim.getObjects(self.sim.handle_all)
            for obj in all_objects:
                try:
                    obj_type = self.sim.getObjectType(obj)
                    if (obj_type == self.sim.object_shape_type and 
                        obj != self.white_ball and 
                        "ball" in self.sim.getObjectAlias(obj, 1).lower()):
                        self.balls.append(obj)
                except:
                    continue
        except:
            print("Error finding other balls")
            
        print(f"Found {len(self.balls)} other balls")
        
        # Cari meja
        self.table = None
        try:
            all_objects = self.sim.getObjects(self.sim.handle_all)
            for obj in all_objects:
                try:
                    name = self.sim.getObjectAlias(obj, 1).lower()
                    if "table" in name or "plane" in name:
                        self.table = obj
                        print(f"Found table: {name}")
                        break
                except:
                    continue
        except:
            print("Error finding table")
    
    def setup_table_parameters(self):
        """Setup parameter meja biliar"""
        # Default values
        self.table_length = 2.0
        self.table_width = 1.0
        self.table_height = 0.03
        
        # Try to get actual table dimensions
        if self.table:
            try:
                self.table_length = self.sim.getObjectFloatParam(self.table, self.sim.objfloatparam_bbox_x)
                self.table_width = self.sim.getObjectFloatParam(self.table, self.sim.objfloatparam_bbox_y)
                print(f"Table dimensions: {self.table_length} x {self.table_width}")
            except:
                print("Using default table dimensions")
        
        # Define pocket positions
        self.pockets = [
            [-self.table_length/2, -self.table_width/2, self.table_height],  # Bottom left
            [-self.table_length/2, self.table_width/2, self.table_height],   # Top left
            [self.table_length/2, -self.table_width/2, self.table_height],   # Bottom right
            [self.table_length/2, self.table_width/2, self.table_height],    # Top right
            [0, -self.table_width/2, self.table_height],                     # Bottom middle
            [0, self.table_width/2, self.table_height]                       # Top middle
        ]
        
        self.pocket_radius = 0.06  # Pocket radius in meters
    
    def reset_white_ball(self):
        """Reset posisi bola putih ke posisi awal"""
        try:
            self.sim.setObjectPosition(self.white_ball, self.sim.handle_world, self.white_ball_start_pos)
            self.sim.setObjectQuaternion(self.white_ball, self.sim.handle_world, [0, 0, 0, 1])
            self.sim.setVelocity(self.white_ball, [0, 0, 0], [0, 0, 0])
            print("White ball reset to starting position")
        except Exception as e:
            print(f"Error resetting white ball: {e}")
    
    def check_pocket(self, ball_handle):
        """Periksa apakah bola masuk ke kantong"""
        try:
            ball_pos = self.sim.getObjectPosition(ball_handle, self.sim.handle_world)
            
            for pocket in self.pockets:
                distance = math.sqrt((ball_pos[0]-pocket[0])**2 + 
                                     (ball_pos[1]-pocket[1])**2 + 
                                     (ball_pos[2]-pocket[2])**2)
                if distance < self.pocket_radius:
                    return True
            return False
        except:
            return False
    
    def get_ball_velocity(self, ball_handle):
        """Dapatkan kecepatan bola"""
        try:
            linear_vel, angular_vel = self.sim.getVelocity(ball_handle)
            speed = math.sqrt(linear_vel[0]**2 + linear_vel[1]**2 + linear_vel[2]**2)
            return speed
        except:
            return 0
    
    def all_balls_stopped(self):
        """Periksa apakah semua bola sudah berhenti"""
        # Periksa bola putih
        if self.get_ball_velocity(self.white_ball) > 0.01:
            return False
        
        # Periksa bola lainnya
        for ball in self.balls:
            if self.get_ball_velocity(ball) > 0.01:
                return False
        
        return True
    
    def apply_force_and_torque(self, force_magnitude, force_angle, torque_magnitude, torque_angle):
        """Terapkan gaya dan torsi pada bola putih"""
        try:
            # Konversi sudut ke radian
            force_angle_rad = math.radians(force_angle)
            torque_angle_rad = math.radians(torque_angle)
            
            # Hitung komponen gaya (dalam bidang XY)
            fx = force_magnitude * math.cos(force_angle_rad)
            fy = force_magnitude * math.sin(force_angle_rad)
            fz = 0  # Tidak ada gaya vertikal
            
            # Hitung komponen torsi (dalam bidang XY, menghasilkan spin pada sumbu Z)
            tx = torque_magnitude * math.cos(torque_angle_rad)
            ty = torque_magnitude * math.sin(torque_angle_rad)
            tz = 0  # Torsi vertikal diatur ke 0
            
            # Terapkan gaya
            self.sim.addForce(self.white_ball, [0, 0, 0], [fx, fy, fz])
            
            # Terapkan torsi
            self.sim.addTorque(self.white_ball, [tx, ty, tz])
            
            print(f"Applied force: {force_magnitude:.2f}N at {force_angle:.1f}°")
            print(f"Applied torque: {torque_magnitude:.2f}Nm at {torque_angle:.1f}°")
            
            return True
        except Exception as e:
            print(f"Error applying force/torque: {e}")
            return False
    
    def get_user_input(self):
        """Dapatkan input dari pengguna"""
        print("\n=== Input Shot Parameters ===")
        print("Enter values for force and torque (or 'quit' to exit)")
        
        try:
            # Input gaya
            force_input = input("Force magnitude (N, 0-50): ")
            if force_input.lower() == 'quit':
                return None, None, None, None
            
            force_magnitude = float(force_input)
            if force_magnitude < 0 or force_magnitude > 50:
                print("Force must be between 0 and 50N")
                return self.get_user_input()
            
            # Input sudut gaya
            angle_input = input("Force angle (degrees, 0-360): ")
            if angle_input.lower() == 'quit':
                return None, None, None, None
            
            force_angle = float(angle_input)
            
            # Input torsi
            torque_input = input("Torque magnitude (Nm, 0-10): ")
            if torque_input.lower() == 'quit':
                return None, None, None, None
            
            torque_magnitude = float(torque_input)
            if torque_magnitude < 0 or torque_magnitude > 10:
                print("Torque must be between 0 and 10Nm")
                return self.get_user_input()
            
            # Input sudut torsi
            torque_angle_input = input("Torque angle (degrees, 0-360): ")
            if torque_angle_input.lower() == 'quit':
                return None, None, None, None
            
            torque_angle = float(torque_angle_input)
            
            return force_magnitude, force_angle, torque_magnitude, torque_angle
            
        except ValueError:
            print("Invalid input. Please enter numeric values.")
            return self.get_user_input()
        except KeyboardInterrupt:
            return None, None, None, None
    
    def run_game(self):
        """Jalankan loop permainan utama"""
        print("\n=== Billiard Game ===")
        print("Controls: Enter force and torque values to shoot the white ball")
        print("Type 'quit' at any prompt to exit")
        
        # Reset bola putih ke posisi awal
        self.reset_white_ball()
        
        while self.game_active:
            # Dapatkan input dari pengguna
            force, force_angle, torque, torque_angle = self.get_user_input()
            
            # Keluar jika pengguna memasukkan 'quit'
            if force is None:
                break
            
            # Terapkan gaya dan torsi
            if self.apply_force_and_torque(force, force_angle, torque, torque_angle):
                # Tunggu hingga semua bola berhenti
                print("Waiting for balls to stop...")
                
                pocketed_balls = []
                while not self.all_balls_stopped():
                    # Periksa apakah ada bola yang masuk ke kantong
                    for i, ball in enumerate(self.balls):
                        if ball not in pocketed_balls and self.check_pocket(ball):
                            print(f"Ball {i+1} pocketed!")
                            pocketed_balls.append(ball)
                            # Pindahkan bola yang masuk ke bawah meja
                            self.sim.setObjectPosition(ball, self.sim.handle_world, [0, 0, -1])
                    
                    # Periksa apakah bola putih masuk ke kantong
                    if self.check_pocket(self.white_ball):
                        print("White ball pocketed! Resetting...")
                        self.reset_white_ball()
                        break
                    
                    # Langkah simulasi
                    self.sim.step()
                    time.sleep(0.01)
                
                print("All balls have stopped")
                
                # Reset bola putih jika masih di atas meja
                if not self.check_pocket(self.white_ball):
                    self.reset_white_ball()
            
        # Berhenti saat keluar
        self.sim.stopSimulation()
        print("Game ended")

# Jalankan permainan
if __name__ == "__main__":
    game = BilliardGame()
    game.run_game()
