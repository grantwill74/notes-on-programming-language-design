```haskell
hasTenth :: [a] -> Bool
hasTenth = (>= 11) . length

swapPair :: (a, b) -> (b, a)
swapPair (x, y) = (y, x)

everyOther :: (a -> a) -> [a] -> [a]
everyOther f [] = []
everyOther f [x] = [f x]
everyOther f (x : y : xs) = f x : y : everyOther f xs

swapEvens :: [a] -> [a] -> ([a], [a])
swapEvens x y = unzip $ everyOther swapPair $ zip x y
```