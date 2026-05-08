How to change Php versions?
brew unlink php          # if you have “php” linked

brew unlink php@8.2      # unlink any other you might have linked

brew link --force --overwrite php@7.4
