MacOS how to create unlimited history like Linux

Terminal:
cat << 'EOF' >> ~/.zshrc

# Increase macOS Terminal history size
export HISTSIZE=100000
export SAVEHIST=100000

# Save commands immediately instead of waiting for the terminal to close
setopt INC_APPEND_HISTORY
# Share history across multiple terminal tabs
setopt SHARE_HISTORY
EOF