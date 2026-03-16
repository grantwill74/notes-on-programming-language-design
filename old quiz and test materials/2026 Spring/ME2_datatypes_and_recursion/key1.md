
```haskell
-- 1.
data Florp = Flip Int | Flop Int Int 

-- 2.
f :: Int -> Florp
f n
    | even n = Flop n n
    | otherwise = Flip n

-- 3.
everyThird :: [a] -> [a]
everyThird (x : _ : _ : xs) = x : everyThird xs
everyThird (x : _) = [x]
everyThird [] = []

-- 4.
stripEmpty :: [String] -> [String]
stripEmpty [] = []
stripEmpty ([] : xs) = stripEmpty xs 
stripEmpty (x : xs) = x : stripEmpty xs
```