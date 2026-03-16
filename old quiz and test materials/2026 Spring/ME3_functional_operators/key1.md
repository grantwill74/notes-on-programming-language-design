```haskell
-- problem 1
whichEven :: [Int] -> [Bool]
whichEven = map even
-- alternatively, if you forgot about the "even" built-in function
whichEven = map (== 0) . map (`mod` 2)

-- problem 2
any :: [Bool] -> Bool
any = foldl (||) False

-- problem 3
anyEven :: [Int] -> Bool
anyEven = any . whichEven

-- problem 4
noEven :: [Int] -> Bool
noEven = not . anyEven
```