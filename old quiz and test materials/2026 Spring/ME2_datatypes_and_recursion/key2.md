```haskell
-- 1.
maybeHeadPair :: [a] -> Maybe (a, a)
maybeHeadPair (x : y : _) = Just (x, y)
maybeHeadPair _ = Nothing
-- 2.
pushPair :: (a, a) -> [a] -> [a]
pushPair (x, y) l = x : y : l
-- 3.
safeDiv :: Int -> Int -> Maybe Int
safeDiv x 0 = Nothing
safeDiv x y = Just $ div x y
-- 4.
isNothing' :: Maybe a -> Bool
isNothing' Nothing = True
isNothing' _ = False
-- or if you're feeling clever, this technically follows the test rules
isNothing' :: Maybe a -> Bool
isNothing' = isNothing
```