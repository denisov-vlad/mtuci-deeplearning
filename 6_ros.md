# ROS


Работа выполняется в операционной системе Ubuntu 16.04.

Далее необходимо установить библиотеку Robotic Operating System (ROS). Была выбрана версия ROS Kinetic, 
т.к. используется система Ubuntu 16.04 (Xenial). 

Данная библиотека предоставляет удобные обертки всех 
используемых библиотек, а также предлагает хорошую инфраструктуру для быстрого создания прототипа.

Для установки необходимо дать системе доступ к репозиториям с данной библиотекой. Делается это следующими командами:
```bash
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key 421C365BD9FF1F717815A3895523BAEEB01FA116
```
Затем необходимо обновить систему и установить ROS:
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install ros-kinetic-desktop-full
```

Перед использованием ROS, необходимо будет инициализировать `rosdep`. Rosdep позволяет легко устанавливать 
зависимости системы для источника, который нужно скомпилировать, и требуется для запуска некоторых 
основных компонентов в ROS:
```bash
sudo rosdep init
rosdep update
```
Будет удобно, если переменные среды ROS будут автоматически добавляться в сессию bash каждый раз при запуске новой оболочки:
```bash
echo "source /opt/ros/kinetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```
`Rosinstall` - это часто используемый инструмент командной строки в ROS, который распространяется отдельно. 
Он позволяет легко загружать множество исходных деревьев для пакетов ROS с помощью одной команд. 
Чтобы установить этот инструмент на Ubuntu нужно сделать следующее:
```bash
sudo apt-get install python-rosinstall
```
На этом установка ROS закончена.
ROS установлена, теперь нужно настроить workspace. Workspace представляет собой папку, в которой 
размещаются пакеты ROS, собираемые из исходников.
ROS использует две системы сборки: rosbuild и catkin. В настоящий момент подавляющее большинство 
пакетов уже мигрировали на систему сборки catkin, но некоторые используют старенькую rosbuild. LSD SLAM относится именно к ним (хотя в репозитории и есть ветки, в которых вроде бы произведена миграция на catkin, по факту же там лежат битые скрипты, а pull request на исправление этих ошибок не принят). Поэтому потратим немного времени и настроим workspace для обеих систем сборки.
Создадим папку ros_workspace внутри домашней папки
```bash
mkdir ~/ros_workspace
```
Настройка catkin:
```bash
mkdir –p ~/ros_workspace/catkin/src && cd $_
catkin_init_workspace
cd .. && catkin_make
```

Настройка rosbuild:
```bash
mkdir ~/ros_workspace/rosbuild && cd $_
rosws init . ~/ros_workspace/catkin/devel
mkdir packages
rosws set ~/ros_workspace/rosbuild/packages -t .
echo "source ~/ros_workspace/rosbuild/setup.bash" >> ~/.bashrc
```

Далее необходимо перезапустить консоль, чтобы изменения вступили в силу. Для проверки того, что все настроено, 
корректно выполним команду: `roscd`.
Вы должны перейти в папку `~/ros_workspace/rosbuild`.
Теперь установим необходимые для SLAM пакеты. Для монокулярного SLAM мы рассмотрим две реализации: PTAM и LSD SLAM.
Установка LSD SLAM:
```bash
sudo apt-get install ros-kinetic-libg2o liblapack-dev libblas-dev freeglut3-dev libqglviewer-dev libsuitesparse-dev libx11-dev
cd ~/ros_workspace/rosbuild/package
git clone https://github.com/tum-vision/lsd_slam.git
rosmake lsd_slam
```

Также необходимо включить поддержку обнаружения циклов (loop closure). Для этого необходимо отредактировать 
файл `lsd_slam_core/CmakeLists.txt`, раскомментировав следующие строки:
```text
add_subdirectory(${PROJECT_SOURCE_DIR}/thirdparty/openFabMap)
include_directories(${PROJECT_SOURCE_DIR}/thirdparty/openFabMap/include)
add_definitions("-DHAVE_FABMAP")
set(FABMAP_LIB openFABMAP)
```

Установка PTAM и RTAB-Map:
```bash
cd ~/ros_workspace/catkin/src
git clone https://github.com/ethz-asl/ethzasl_ptam
cd .. && catkin_make
sudo apt-get install ros-kinetic-rtabmap-ros
```

Далее необходимо поставить пакет usb-cam. Пакет usb-cam необходим для использования обычной RGB USB камеры 
в качестве источника данных:
```bash
cd ~/ros_workspace/catkin/src
git clone https://github.com/bosch-ros-pkg/usb_cam
cd.. && catkin_make
sudo apt-get install v4l-utils
```
Проверим работоспособность камеры – подключите камеру и посмотрите, какой идентификатор устройства был ей присвоен:
```bash
ls /dev/video*
```

В нашем случае это video0. Далее перейдем в папку (предварительно создав ее) `~/ros_workspace/test` и создадим 
в ней текстовый файл camera.launch запуска со следующим содержанием:
```text
<launch>
        <node pkg="usb_cam" type="usb_cam_node" name="camera" output="screen">
                <param name="video_device" value="/dev/video0"/>
                <param name="image_width" value="320"/>
                <param name="image_height" value="240"/>
        </node>

        <node pkg="image_view" type="image_view" name="viewer">
                <remap from="image" to="/camera/image_raw"/>
        </node>
</launch>
```

С помощью данного файла мы запускаем два узла – usb_cam_node из пакета usb_cam – для получения изображения 
с нашей камеры. И image_view из одноименного пакета – с помощью этого узла мы отображаем на экране данные, 
полученные с камеры. Так же для узла камеры мы определяем некоторые параметры, самым главным из которых 
является video_device, который соответствует устройству нашей камеры.

Теперь выполняем этот файл с помощью команды (нужно находиться в той же папке, где и файл):
```bash
roslaunch camera.launch
```

*Выводы*

Установлены пакеты для работы со SLAM. Протестирована работа usb-камеры.

