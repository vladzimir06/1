




    using System;
    using System.Collections.Generic;
   
    class Program
    {
        static void Main(string[] args)
        {
            Game game = new Game();//создание модели
            game.Run();
        }
    }

    class Game
    {
        public ulong Score { get; private set; }
        public ulong[,] Board { get; private set; }

        private  int nRows;
        private  int nCols;
        private Random random = new Random();

        public Game()
        {
            Board = new ulong[4, 4];
            nRows = Board.GetLength(0);
            nCols = Board.GetLength(1);
            Score = 0;
        }

        public void Run()
        {
            bool  nachnem = true;
            do
            {
                if (nachnem)
                {
                    PutNewValue();
                }

                Display();

                if (IsDead())
                {
                    {
                        Console.WriteLine("YOU ARE DEAD!!!");
                        break;
                    }
                }

                Console.WriteLine("Use arrow keys to move the tiles. Press Ctrl-C to exit.");
                ConsoleKeyInfo input = Console.ReadKey(true); // BLOCKING TO WAIT FOR INPUT
                Console.WriteLine(input.Key.ToString());

                switch (input.Key)
                {
                    case ConsoleKey.UpArrow:
                        nachnem = Update(Direction.Up);
                        break;

                    case ConsoleKey.DownArrow:
                        nachnem = Update(Direction.Down);
                        break;

                    case ConsoleKey.LeftArrow:
                        nachnem = Update(Direction.Left);
                        break;

                    case ConsoleKey.RightArrow:
                        nachnem = Update(Direction.Right);
                        break;

                    default:
                        nachnem = false;
                        break;
                }
            }
            while (true); // CTRL-C to break 

            Console.WriteLine("Press any key to quit...");
            Console.Read();
        }

       

        private static bool Update(ulong[,] board, Direction direction, out ulong score)
        {
            int nRows = board.GetLength(0);
            int nCols = board.GetLength(1);

            score = 0;
            bool hasUpdated = false;

            // оверяем умер ли ты в конце Update()

            // оставить строку или столбец true: row; false: column;
            bool isAlongRow = direction == Direction.Left || direction == Direction.Right;

            // внутреняя обработка изменнения индекса
            bool isIncreasing = direction == Direction.Left || direction == Direction.Up;
        // inner - внутренний; outter - внешнийж
            int outterCount = isAlongRow ? nRows : nCols;
            int innerCount = isAlongRow ? nCols : nRows;
            int innerStart = isIncreasing ? 0 : innerCount - 1;
            int innerEnd = isIncreasing ? innerCount - 1 : 0;

            Func<int, int> drop = isIncreasing
                ? new Func<int, int>(innerIndex => innerIndex - 1)
                : new Func<int, int>(innerIndex => innerIndex + 1);

            Func<int, int> reverseDrop = isIncreasing
                ? new Func<int, int>(innerIndex => innerIndex + 1)
                : new Func<int, int>(innerIndex => innerIndex - 1);

            Func<ulong[,], int, int, ulong> getValue = isAlongRow
                ? new Func<ulong[,], int, int, ulong>((x, i, j) => x[i, j])
                : new Func<ulong[,], int, int, ulong>((x, i, j) => x[j, i]);

            Action<ulong[,], int, int, ulong> setValue = isAlongRow
                ? new Action<ulong[,], int, int, ulong>((x, i, j, v) => x[i, j] = v)
                : new Action<ulong[,], int, int, ulong>((x, i, j, v) => x[j, i] = v);

            Func<int, bool> innerCondition = index => Math.Min(innerStart, innerEnd) <= index && index <= Math.Max(innerStart, innerEnd);


            for (int i = 0; i < outterCount; i++)
            {
                for (int j = innerStart; innerCondition(j); j = reverseDrop(j))
                {
                    if (getValue(board, i, j) == 0)
                    {
                        continue;
                    }

                    int newJ = j;
                    do
                    {
                        newJ = drop(newJ);
                    }
                    // продолжаем пока не достигли границы и есть не занятые поля
                    while (innerCondition(newJ) && getValue(board, i, newJ) == 0);

                    if (innerCondition(newJ) && getValue(board, i, newJ) == getValue(board, i, j))
                    {
                        // не достигли границы , нет слияния и все заполнено
                        
                        ulong newValue = getValue(board, i, newJ) * 2;
                        setValue(board, i, newJ, newValue);
                        setValue(board, i, j, 0);

                        hasUpdated = true;
                        score += newValue;
                    }
                    else
                    {
                    // Достигнул границы , попали в узел с другим значением ,мы попали в узел с тем же значением, но произошло предыдущее слияние
                   
                     newJ = reverseDrop(newJ); // вернутся к изначальной позиции
                        if (newJ != j)
                        {
                            
                            hasUpdated = true;
                        }

                        ulong value = getValue(board, i, j);
                        setValue(board, i, j, 0);
                        setValue(board, i, newJ, value);
                    }
                }
            }

            return hasUpdated;
        }

        private bool Update(Direction dir)
        {
            ulong score;
            bool isUpdated = Game.Update(Board, dir, out score);
            Score += score;
            return isUpdated;
        }

        private bool IsDead()
        {
            ulong score;
            foreach (Direction dir in new Direction[] { Direction.Down, Direction.Up, Direction.Left, Direction.Right })
            {
                ulong[,] clone = (ulong[,])Board.Clone();
                if (Game.Update(clone, dir, out score))
                {
                    return false;
                }
            }

            // все пробуем работает или  нет
            return true;
        }

        private void Display()
        {
            Console.Clear();
            Console.WriteLine();
            for (int i = 0; i < nRows; i++)
            {
                for (int j = 0; j < nCols; j++)
                {
                    {
                        Console.Write(string.Format("{0,6}", Board[i, j]));
                    }
                }

                Console.WriteLine();
                Console.WriteLine();
            }

            Console.WriteLine("Score: {0}", Score);
            Console.WriteLine();
        }

        private void PutNewValue()
        {
            //ищем пустые поля
            List<Tuple<int, int>> emptySlots = new List<Tuple<int, int>>();
            for (int iRow = 0; iRow < nRows; iRow++)
            {
                for (int iCol = 0; iCol < nCols; iCol++)
                {
                    if (Board[iRow, iCol] == 0)
                    {
                        emptySlots.Add(new Tuple<int, int>(iRow, iCol));
                    }
                }
            }

            // пока есть свободный слот  еще жив
            int iSlot = random.Next(0, emptySlots.Count); // рандомное заполнение слота
            ulong value = random.Next(0, 100) < 80 ? (ulong)2 : (ulong)4; // рандомно 2 или 4
            Board[emptySlots[iSlot].Item1, emptySlots[iSlot].Item2] = value;
        }

        #region Utility Classes
        enum Direction
        {
            Up,
            Down,
            Right,
            Left,
        }

     
            public void Dispose()
            {
                Console.ResetColor();
            }
        }
        #endregion Utility Classes
    
