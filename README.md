import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from turtlesim.msg import Pose
from turtlesim.srv import TeleportAbsolute, Clear
import math
import time
import subprocess

class DrawInitials(Node):
    def __init__(self):
        super().__init__('draw_initials')
        
        self.publisher = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.pose_subscriber = self.create_subscription(Pose, '/turtle1/pose', self.pose_callback, 10)
        
        self.current_pose = None
        self.pose_received = False
        
        self.get_logger().info('Ожидание получения позиции черепашки...')
        for _ in range(50):
            rclpy.spin_once(self, timeout_sec=0.1)
            if self.pose_received:
                break
        
        if not self.pose_received:
            self.get_logger().error('Не удалось получить позицию черепашки')
            return
        
        self.get_logger().info('Начинаем рисовать инициалы A.B.')
        self.clear_screen()
        time.sleep(0.5)
        self.draw_initials()
    
    def pose_callback(self, msg):
        self.current_pose = msg
        self.pose_received = True
    
    def move(self, distance, speed=1.0):
        if self.current_pose is None:
            return
        twist = Twist()
        twist.linear.x = speed
        start_x = self.current_pose.x
        start_y = self.current_pose.y
        while rclpy.ok():
            rclpy.spin_once(self, timeout_sec=0.05)
            if self.current_pose is None:
                continue
            current_distance = math.sqrt((self.current_pose.x - start_x)**2 + 
                                        (self.current_pose.y - start_y)**2)
            if current_distance >= distance:
                break
            self.publisher.publish(twist)
        self.stop()
        time.sleep(0.3)
    
    def turn(self, angle, speed=1.0):
        if self.current_pose is None:
            return
        twist = Twist()
        twist.angular.z = speed if angle > 0 else -speed
        start_angle = self.current_pose.theta
        target_angle = start_angle + math.radians(angle)
        while rclpy.ok():
            rclpy.spin_once(self, timeout_sec=0.05)
            if self.current_pose is None:
                continue
            current_angle = self.current_pose.theta
            angle_diff = target_angle - current_angle
            while angle_diff > math.pi:
                angle_diff -= 2 * math.pi
            while angle_diff < -math.pi:
                angle_diff += 2 * math.pi
            if abs(angle_diff) < 0.05:
                break
            self.publisher.publish(twist)
        self.stop()
        time.sleep(0.3)
    
    def stop(self):
        twist = Twist()
        twist.linear.x = 0.0
        twist.angular.z = 0.0
        self.publisher.publish(twist)
    
    def draw_A(self):
        self.get_logger().info('Рисуем букву A')
        self.turn(-60)
        self.move(3.0, speed=0.8)
        self.turn(120)
        self.move(3.0, speed=0.8)
        self.turn(-60)
        self.move(1.5, speed=0.6)
        self.turn(-60)
        self.move(2.0, speed=0.6)
        self.turn(120)
        self.move(1.5, speed=0.6)
        self.turn(60)
        self.move(0.5, speed=0.3)
    
    def draw_B(self):
        self.get_logger().info('Рисуем букву B')
        self.turn(90)
        self.move(3.0, speed=0.8)
        self.turn(-90)
        self.move(1.2, speed=0.6)
        self.turn(-90)
        self.move(1.2, speed=0.6)
        self.turn(-90)
        self.move(1.2, speed=0.6)
        self.turn(90)
        self.move(1.2, speed=0.6)
        self.turn(90)
        self.move(1.2, speed=0.6)
        self.turn(90)
        self.move(1.2, speed=0.6)
    
    def draw_dot(self):
        self.get_logger().info('Ставим точку')
        self.move(0.2, speed=0.2)
        time.sleep(0.2)
        self.move(0.2, speed=0.2)
    
    def teleport(self, x, y):
        client = self.create_client(TeleportAbsolute, '/turtle1/teleport_absolute')
        while not client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('Ждем сервис телепортации...')
        request = TeleportAbsolute.Request()
        request.x = x
        request.y = y
        request.theta = 0.0
        future = client.call_async(request)
        rclpy.spin_until_future_complete(self, future)
        time.sleep(0.3)
        rclpy.spin_once(self, timeout_sec=0.1)
        if self.current_pose:
            self.current_pose.x = x
            self.current_pose.y = y
            self.current_pose.theta = 0.0
    
    def clear_screen(self):
        client = self.create_client(Clear, '/clear')
        while not client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('Ждем сервис очистки...')
        request = Clear.Request()
        client.call_async(request)
        self.get_logger().info('Экран очищен')
    
    def draw_initials(self):
        try:
            self.teleport(3.0, 7.0)
            time.sleep(0.5)
            self.draw_A()
            self.teleport(5.0, 6.5)
            self.draw_dot()
            self.teleport(6.5, 7.0)
            self.draw_B()
            self.teleport(9.5, 6.5)
            self.draw_dot()
            self.get_logger().info('Рисование завершено! Инициалы A.B. нарисованы!')
        except Exception as e:
            self.get_logger().error(f'Ошибка при рисовании: {e}')

def main(args=None):
    rclpy.init(args=args)
    try:
        node = DrawInitials()
        rclpy.spin_once(node, timeout_sec=1.0)
    except:
        print("Запуск turtlesim_node...")
        subprocess.Popen(['ros2', 'run', 'turtlesim', 'turtlesim_node'])
        time.sleep(3)
        node = DrawInitials()
    for _ in range(50):
        rclpy.spin_once(node, timeout_sec=0.1)
    node.destroy_node()
    rclpy.shutdown()
    print("Программа завершена")

if __name__ == '__main__':
    main()
