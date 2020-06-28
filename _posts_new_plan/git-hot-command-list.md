깃 명령어, git 명령어, git command


로컬 브랜치 만들기$ git checkout -b <branch-name>

로컬 브랜치 원격 올리기$ git push <remote-name> <branch-name>

원격 브랜치 로컬로 가져오기$ git checkout -t <remote-name>/<branch-name>

원격 브랜치 로컬 브랜치에서 트래킹(연동)$ git branch --set-upstream-to <remote-name>/<branch-name>

로컬 브랜치 삭제하기$ git branch -d <branch-name>, git branch -D <branch-name>

원격 브랜치 삭제하기$ 로컬 브랜치 삭제 후 git push <remote-name> :<branch-name>, git push <remote-name> -d <branch-name>

원격 브랜치 리스트$ git branch -r

로컬 브랜치 리스트$ git branch

모든 브랜치 리스트$ git branch -a

로컬에서 원격 브랜치 리스트 갱신$ git remote update, git fetch

특정 커밋 시점으로 되돌리기$ git reset --soft <commit-hash>, git reset --hard <commit-hash>