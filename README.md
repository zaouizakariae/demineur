# Minesweeper

# INTRODUCTION

Minesweeper is a single-player puzzle video game. The objective of the game is to clear a rectangular board containing hidden "mines" or bombs without detonating any of them, with help from clues about the number of neighbouring mines in each field.

![Capture d’écran 2022-02-07 001507](https://user-images.githubusercontent.com/85891554/152705697-be3710a8-6c9c-49be-8815-42e2a15bfe77.png) 
![Capture d’écran 2022-02-07 001524](https://user-images.githubusercontent.com/85891554/152705674-b46924ce-5b67-4242-8b0d-366692b403ac.png)

we are going to use qt designer to create our interface 
our interface is composed of two buttons , lcd number and a QGridLayout container

![Capture d’écran 2022-02-07 003256](https://user-images.githubusercontent.com/85891554/152706468-a119acef-5c15-4636-be78-de8ce16bc8d0.png)

# Cell

fisrt thing we need  is to create a cell class with whom we are going to fill our container

- we need to set the statut of each cell if its a flag , bomb ect based on the user actions
- 1 the constructor must include this code
```cpp
   cell::cell(bool bomb, QWidget * parent) : QPushButton(parent)
   {
    bombCount=0;
    flagged=false;
    boxState=BoxState::unclicked;
    setBomb(bomb);
    update();
   }
```
> the constructor takes a boolean attribute named bomb wich is genarated randomly in the mainwindow class
- 2 now we need a bomb setter and a function that returns that a cell is clicked
```cpp
    void cell::setBomb(bool b){
         bomb=b;
    }
    bool cell::isClicked(){
    return (boxState==BoxState::clicked);
    }
```
- 3 if a cell is clicked we need to know if its a bomb or not and update 
```cpp
   bool cell::boxClicked(){


    if(bomb){
        boxState=BoxState::exploded;
    }else{
        boxState=BoxState::clicked;
    }
    update();
    return bomb;
   }
```
- 4 if a cell is right clicked it turns into a flag
```cpp
   bool cell::boxRightClicked(){
   
    setStyleSheet("QPushButton {image: url(:/flag.png);}");
    flagged=true;
    return bomb;
   }
```
- 5 we create a function that returns how many bombs are around the box 
```cpp
void cell::setBombCount(int c){
    if(c!=0)setText(QString::number(c));
}
```
- 6 ovveride a mouse press event that emits a signal if the cell is right clicked or left clicked 
```cpp
   void cell::mousePressEvent(QMouseEvent *e)
   {
    if(e->button()==Qt::RightButton)
        emit rightClicked();
    if(e->button()==Qt::LeftButton && !flagged)
        emit clicked();
   }
```
- 7 now the update function , it changes the style of a cell based on the player actions
```cpp
  void cell::update(){
    QString style="";
    if(boxState==BoxState::unclicked){

       style+="QPushButton { image: url(:/box.png);border-style: inset;border:0px;}";
        setText("");
    }else if(boxState==BoxState::clicked){
        style+="QPushButton {background-color: rgb(220,220,220);border-style: inset;}";
        setEnabled(false);
    }else if(boxState==BoxState::exploded){
         style+="QPushButton {image: url(:/bomb.png);border-style: inset;}";
        setEnabled(false);
    }
    setStyleSheet(style);

}
```
> to know the state of each cell we created an  enum class called BoxState that contains the different unchangeable states of the box
```cpp
enum class  BoxState
{
    unclicked, clicked, exploded
};

```
# setup container

now lets setup our container 
- we use QSignalMapper to create a left and right signal that are going to  connect left and right clicked actions to the cells
- we used a randbool fucntion to change randomly the state of a cell to a bomb  
  ```cpp
  bool MainWindow::randbool(int w){
    return (rand()%w)==0;
  }
  ```
- we added some style to our cells
```cpp
void MainWindow::setupContainer(){

    QSignalMapper *Left;
    QSignalMapper *Right;
    Left = new QSignalMapper(this);
   Right = new QSignalMapper(this);
    connect(Left, SIGNAL(mapped(int)),this, SLOT(boxL(int)));
    connect(Right, SIGNAL(mapped(int)),this, SLOT(boxR(int)));
    for(int i = 0 ; i < 10 ; i++){
        for(int j = 0 ; j < 10 ; j++){
          bool bomb=randbool(5);
            grid[i][j] = new cell(bomb);
            auto bobo = grid[i][j];
            
             bobo->setSizePolicy(QSizePolicy ::Expanding , QSizePolicy ::Expanding );
              bobo->setFocusPolicy(Qt::NoFocus);
              bobo->setMinimumSize(40,40);
              bobo->setContentsMargins(0,0,0,0);
              bobo->setMaximumSize(40,40);
            QFont f =bobo->font();
            f.setPointSize(19);
            bobo->setFont(f);
            ui->container->addWidget(bobo,i,j);
            Left->setMapping(bobo,(i*1000)+j);
            Right->setMapping(bobo,(i*1000)+j);
            connect(bobo,SIGNAL(clicked()),Left,SLOT(map()));
            connect(bobo,SIGNAL(rightClicked()),Right,SLOT(map()));

        }
    }
}
```
# leftclicked action
- if a box is left clicked , we check if its  a bomb so we execute the bigbang function that explodes all the bombs or we need to set a bomb count of how many are around the box, and of course we check if we won
```cpp
void MainWindow::boxL(int i){
    click.play();
    if(t==0)
        timer->start(1000);
    if(!isGameOngoing)return;
    int p[2];
    p[0]=i/1000;
    p[1]=i%1000;

    cell* bobo=grid[p[0]][p[1]];
    if(bobo->bomb){
        bigbang();
        bobo->boxClicked();
        return;
    }
    bobo->boxClicked();
    clear(p[0],p[1]);
    winCheck();

}
```
# rightclicked action
- for the right clicked Slot we set the cell status to flagged 
```cpp
void MainWindow::boxR(int id){
    click.play();
    if(t==0)
        timer->start(1000);
    if(!isGameOngoing)return;
    int p[2];
    p[0]=id/1000;
    p[1]=id%1000;

    cell* b=grid[p[0]][p[1]];
    b->boxRightClicked();
    winCheck();
}
```
# clear the empty cells
- to clear the empty cells we used a recursive way that checks the states of each cell around our cell and examine if they are empty or not
```cpp
void MainWindow::clear(int x,int y){
    grid[x][y]->boxClicked();
    grid[x][y]->setBombCount(findBombCount(x,y));
    if(findBombCount(x,y)==0){
        for(int i=-1;i<=1;i++){
            if(x+i<0 || x+i>10-1)continue;
            for(int j=-1;j<=1;j++){
                if(y+j<0 || y+j>10-1 || (i==0 && j==0))continue;
                if(!grid[x+i][y+j]->isClicked()){
                    clear(x+i,y+j);
                }
            }
        }

    }else{
        return;
    }
}
```
- now the implementation of the function that finds out how many bombs are around a specific cell
```cpp
int MainWindow::findBombCount(int x,int y){
    int count=0;
    for(int i=-1;i<=1;i++){
        if(x+i<0 || x+i>9)
            continue;
        for(int j=-1;j<=1;j++){
            if(y+j<0 || y+j>9 || (i==0 && j==0))
                continue;

            cell* cell=grid[x+i][y+j];
            if(cell->bomb)count++;
        }
    }
    return count;
}
```
# bigbang
- this funtion is going to explode all the bombs in our map and its executed when a box that contains abomb is clicked
```cpp
void MainWindow::bigbang(){

    lost.play();
    ui->pp->setStyleSheet("QPushButton { image: url(:/dead.png);background: transparent;}");
    for(int i=0;i<10;i++){
        for(int j=0;j<10;j++){
            cell *bobo=grid[i][j];
            if(bobo->bomb){
                bobo->boxClicked();
            }

        }
    }
    timer->stop();

    isGameOngoing=false;
}
```
> now that the value of  isGameOngoing=false if we click on a cell nothing will change until we start a new game using the dead face button
# win check
- to check if we won or not we need to be sure that all the  boxes that do not include bombs are  clicked and thhat the cells that are flagged are bombs
```cpp
bool MainWindow::winCheck(){
    for(int i=0;i<10;i++){
        for(int j=0;j<10;j++){
            cell *bobo=grid[i][j];

            if(!bobo->bomb && !bobo->isClicked())return false;
            if(!bobo->bomb && bobo->flagged)return false;
        }
    }
 lost.setMedia(QUrl("qrc:/w.mp3"));
 lost.play();
    isGameOngoing=false;
    return true;
}
```
# timer
- in this game the score is the time it took us to complete the game
  - we need to connect a timer to a slot that shows the time on our lcd number
  ```cpp
  connect(timer, &QTimer::timeout, this, &MainWindow::showTime);
  ```
  > showtime slot
  ```cpp
  void MainWindow::showTime()
  {
    t++;
    ui->lcdNumber_2->display(t);
  }
  ```
  - this timer start after the first action so we need to add this code to the first left or right clicked action
  ```cpp
  if(t==0)
        timer->start(1000);
  ```
  - and it stops if we click on a bomb , we add this line of code to the end of the bigbang function
  ```cpp
  timer->stop();
  ```
# sound effects
- we added a song to the game using QMediaPlayer
```cpp
QMediaPlayer sing;
```
- now in our constructor we need to add this code so the sound start
```cpp
sing.setMedia(QUrl("qrc:/classic.mp3"));
sing.play();
```
- to stop the song we have to click on the mute button wich is connected to the cms slot
```cpp
void MainWindow::cms(){
    a= !a;
    if(a){
        sing.play();
        ui->pp1->setStyleSheet("QPushButton { image: url(:/sound.png);background: transparent;}");
    }else{
        sing.stop();
        ui->pp1->setStyleSheet("QPushButton { image: url(:/mute.png);background: transparent;}");
    }

}
```
- we are going to do the same thing for the win , lost  and click sounds
# new game
- to start a new we need to click on the smiley face ih the middle of our game
- this button is connected to the newg slot and it turns to a dead face if we lose the game
```cpp
void MainWindow::newg(){
     lost.setMedia(QUrl("qrc:/lost.mp3"));
    timer->stop();
    r=0;
    t=0;
    ui->pp->setStyleSheet("QPushButton { image: url(:/smile.png);background: transparent;}");

    setupContainer();
    isGameOngoing=true;
}

```
# test
https://user-images.githubusercontent.com/85891554/152710989-926347fe-2eea-45bf-9543-1f1d6fc46bf2.mp4

